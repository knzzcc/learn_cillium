k8s 容器除錯思路
具體幾個思路展開，不透過PS aux怎麼找到容器PID 有可能Kubelet掛了pod還在?

下午5:35
好問題，分幾個面向聊聊。

不透過 ps aux 找容器 PID
有幾條路可以走：

從 container runtime 拿：最直接的方式是透過 runtime 本身查。如果是 containerd，用 crictl inspect <container-id> 會回傳 JSON，裡面的 pid 欄位就是容器 init process 的 PID。Docker 的話就是 docker inspect --format '{{.State.Pid}}' <container>。

從 cgroup 反查：到 /sys/fs/cgroup/ 底下找對應的 cgroup 目錄（通常路徑包含 container ID），讀 cgroup.procs 就能拿到該 cgroup 裡所有 process 的 PID 列表。這招在 runtime CLI 壞掉時特別有用。

從 /proc 遍歷：每個 process 的 /proc/<pid>/cgroup 會記錄它屬於哪個 cgroup，所以可以 grep container ID 來反查。類似地，/proc/<pid>/ns/ 底下的 namespace inode 也能用來判斷哪些 process 共享同一個 network/pid namespace，藉此歸類出同一個 pod 的 processes。

從 PID namespace 切入：如果你知道某個 container 的 PID，可以用 nsenter -t <pid> -p -r ps 從那個 namespace 內部看 process tree，不依賴 host 上的 ps。

Kubelet 掛了但 Pod 還在？
完全可能，而且這是理解 K8s 架構的一個關鍵點。

Kubelet 本質上只是一個管理者，它透過 CRI 叫 containerd（或其他 runtime）去啟動 container。一旦 container 跑起來了，它就是 host 上一個普通的 linux process，由 kernel 管理。Kubelet 掛掉只是失去了「管理層」，container process 本身不會被 kill。

實際影響是：

Node 變 NotReady：kubelet 不再向 API server 回報心跳，control plane 會標記 node 為 NotReady。但這只是 control plane 的狀態認知改變，不是實際去殺 process。

Pod 還在跑但沒人管：container 照常運行、照常服務流量（只要 kube-proxy / iptables 規則還在）。但 liveness probe 沒人執行了，所以如果 app 其實已經 hang 住，不會有人幫你重啟它。

驅逐是「邏輯驅逐」：control plane 過了 tolerationSeconds 後會把 pod 標記為要驅逐，但實際的 delete 動作需要 kubelet 去執行。kubelet 掛了，就沒人去真正 stop container。所以你會看到 API server 上 pod 被標 Terminating，但 node 上 container 還活著。

kubelet 恢復後的行為：kubelet 重啟後會做 reconciliation——它會跟 runtime 對帳，發現哪些 container 還在跑，跟 API server 的期望狀態做比對，該殺的殺、該起的起。

一個常見的除錯場景：kubelet OOM 被 kernel kill 掉，但 node 上幾十個 pod 全部照常跑，只是 kubectl 那端看到 node NotReady。這時 ssh 進去會發現 crictl ps 一切正常，只是 systemctl status kubelet 是 dead。

甚麼時候需要這一個?，不透過PS aux怎麼找到容器PID 有可能Kubelet掛了pod還在?

下午5:36
很實際的問題——什麼場景下你會需要「繞過 ps aux 找 PID」？

最常見的場景
容器內沒有 ps 指令：很多 production image 是 distroless 或 alpine 精簡版，根本沒裝 procps。你 kubectl exec 進去發現連 ps 都沒有，這時候只能從 host 側下手。

需要從 host 側對容器做操作：比如你要用 strace、gdb、perf、tcpdump attach 到容器裡的 process，這些工具跑在 host 上，你必須先知道 host 視角的 PID 才能 -p <pid> 去 attach。kubectl exec 進容器看到的 PID 1 不是 host 上的 PID 1。

排查殭屍 container：有時候 runtime 狀態跟實際不一致，crictl ps 顯示 container 已經沒了，但實際上 process 還活著佔著 port 或檔案鎖。這時你需要從 cgroup 或 /proc 反查，才能找到那些「不被 runtime 承認但還在跑」的孤兒 process。

排查資源異常：node 上 CPU 或 memory 飆高，你想快速定位是哪個 container 造成的。直接讀 /sys/fs/cgroup/.../cgroup.procs 配合 /proc/<pid>/status 來看，比 ps aux | grep 更精準，因為你是按 cgroup 歸屬來看而不是靠 process name 猜。

kubelet 或 runtime 半死不活：crictl 要透過 CRI socket 跟 containerd 通訊，如果 containerd 卡住（比如正在做大量 image GC），crictl ps 會 hang 住。這時 /proc 和 cgroup filesystem 是 kernel 直接提供的，不依賴任何 daemon，一定能讀。

簡單講就是一個原則
ps aux 和 crictl 都是「透過某個 userspace 程式」問出來的資訊。當那個程式本身有問題、不存在、或者你需要的是 host 側 PID 而不是 namespace 內的 PID 時，就得回到 kernel 提供的原始介面——/proc 和 /sys/fs/cgroup。這是最底層、最不會騙你的資訊來源。

基本上這個過了之後呢? 是不是就容易多了?

下午5:36
對，過了這關之後基本上就是「有 PID 就好辦事」。

拿到 PID 之後你能做的事情一下子全打開了。/proc/<pid>/ 底下什麼都有——fd 看它開了哪些檔案和 socket、net 看網路連線狀態、maps 看 memory mapping、status 看記憶體用量和執行緒數。要抓封包就 nsenter -t <pid> -n tcpdump，要追 syscall 就 strace -p <pid>，要看效能就 perf top -p <pid>。每個工具都吃 PID，所以找到 PID 等於拿到鑰匙。

真正難的部分其實是「找到 PID 之前」那段，因為你要在 K8s 抽象層（pod name）和 Linux 底層（PID）之間建立對應關係。這中間隔了 kubectl、kubelet、CRI、containerd、runc 好幾層，任何一層出問題都可能讓你從上層查不到。一旦你能繞過這些層直接從 kernel 拿到 PID，後面的除錯就回到傳統 Linux troubleshooting 的領域了，工具成熟、資料多、套路也固定。

所以整個 K8s 容器除錯的思路其實就兩段：第一段是「穿透抽象層找到目標 process」，第二段是「用標準 Linux 工具分析問題」。第一段是 K8s 特有的難點，第二段就是老本行了。

甚麼時候會卡容器權限???

下午5:37
這個問題很實戰。你拿到 PID、工具也準備好了，結果一跑就被權限擋住——通常是這幾種情況：

seccomp
這是最常被忽略的。K8s 預設的 RuntimeDefault seccomp profile 會擋掉一堆 syscall。你在 host 上 strace -p <pid> 沒問題，但如果你是從容器內部跑工具，或者你想讓容器內的 process 做某些操作，就會被擋。典型的像 ptrace（strace/gdb 需要的）、bpf（一些進階追蹤工具）、mount 這些都在黑名單上。報錯通常是 EPERM 或直接 Operation not permitted，不會明確告訴你是 seccomp 擋的，所以很難 debug。

Linux capabilities
即使你跑的是 root，container 裡的 root 預設也被砍掉大部分 capabilities。常被卡住的場景：tcpdump 需要 CAP_NET_RAW、mount 需要 CAP_SYS_ADMIN、strace 需要 CAP_SYS_PTRACE、改 network 設定需要 CAP_NET_ADMIN。Pod spec 裡沒加對應 capability 就是做不了，而且報錯跟 seccomp 一樣都是模糊的 permission denied。

user namespace
如果 cluster 啟用了 user namespace（比較新的功能），容器裡的 root（UID 0）映射到 host 上其實是一個普通 user。這時你從 host 側用 nsenter 進去，看起來是 root，但實際上對 host 資源的存取權限是受限的。

AppArmor / SELinux
很多人不知道自己的 node 有跑這些。Ubuntu 預設帶 AppArmor，RHEL/CentOS 預設帶 SELinux。症狀是你明明有 capability、seccomp 也沒擋，但就是 permission denied。看 dmesg 會看到 AppArmor 的 DENIED 或 SELinux 的 avc: denied 訊息，這是唯一的線索。

read-only filesystem
securityContext 裡設了 readOnlyRootFilesystem: true，你想在容器裡裝工具或寫 temp file 都不行。不是權限問題，但症狀很像，常被搞混。

實務上怎麼處理
最快的方式是不要在問題容器裡面搞，直接在 host 上用 nsenter 帶著對應的 namespace 進去操作。因為你是用 host 上的 root 身份跑工具，只是切了 namespace，所以 seccomp、capabilities、AppArmor 這些容器級的限制都不會套用到你身上。這也是為什麼前面講的「先找到 PID」那麼重要——有了 PID 你就可以從 host 側繞過所有容器權限限制。

另一個常見做法是起一個 debug 用的 privileged pod（或用 kubectl debug node），給齊所有權限，專門拿來當工具箱用。但要注意 production cluster 的 PSP 或 PSA 可能會擋 privileged pod。

PSP 或 PSA 這甚麼??

下午5:38
這兩個都是 K8s 裡「管你的 Pod 能拿多少權限」的機制。

PSP（PodSecurityPolicy）
比較舊的做法，K8s 1.25 已經正式移除了。它是一個 cluster 級的資源，admin 定義一組規則，比如「不准跑 privileged container」、「不准用 hostNetwork」、「只能用某些 volume type」。Pod 建立的時候 admission controller 會檢查，不符合就直接擋掉。

它被移除的原因很實際——太難用了。PSP 的生效邏輯跟 RBAC 綁在一起，是看「建立 Pod 的 ServiceAccount 有沒有被授權使用某個 PSP」，而不是看 namespace。一個 cluster 裡有多個 PSP 的時候，到底套用哪一個、優先順序怎麼算，搞到後來連 admin 自己都搞不清楚。

PSA（Pod Security Admission）
PSP 的替代品，從 1.23 開始內建，1.25 之後成為標準做法。概念簡單很多——直接在 namespace 上打 label 就好，定義三個安全等級：

privileged：完全不限制，什麼都能做。通常給系統級 namespace 像 kube-system 用。

baseline：擋掉明顯危險的東西，像 hostPID、hostNetwork、privileged container，但不會太嚴格。大部分正常 workload 都能過。

restricted：最嚴，要求 non-root、drop all capabilities、read-only root filesystem 之類的。

用法就是在 namespace 上加 label，比如 pod-security.kubernetes.io/enforce: baseline，之後這個 namespace 裡建的 Pod 都會被檢查。比 PSP 直覺太多了。

跟你剛才問的情境的關係
就是當你想在 production cluster 起一個 privileged debug pod 的時候，如果那個 namespace 的 PSA 設成 restricted 或 baseline，你的 debug pod 就會被擋掉。這時候要嘛請 admin 暫時調整 label，要嘛用 kubectl debug node/ 走 host 層級的 debug（這個跑在 node 上，不受 namespace 的 PSA 限制）。

所以整個 K8s 容器除錯的思路其實就兩段：第一段是「穿透抽象層找到目標 process」，第二段是「用標準 Linux 工具分析問題」。第一段是 K8s 特有的難點，第二段就是老本行了。

我覺得這一句你說得很好，但我知道，如果是k8s網路呢?還有kubelet kube-proxy這些組件出錯
又或者說，k8s常見再那些環節會出問題 我以前遇過kmem的問題

下午5:41
你提到的很關鍵，我之前那句話確實只涵蓋了「容器內 process 層級的除錯」。但 K8s 的問題遠不止於此，展開來講大概有這幾個層面。

網路層——最痛的地方
K8s 網路問題特別難是因為它疊了太多層。一個封包從 Pod A 到 Pod B，中間經過 veth pair、bridge 或直接路由（看 CNI）、iptables/ipvs 規則（kube-proxy 管的）、可能還有 overlay（VXLAN/Geneve）。任何一層出問題，症狀都是「連不上」，但根因完全不同。

kube-proxy 出問題的典型症狀是 Service ClusterIP 不通但 Pod IP 直連是通的。因為 kube-proxy 負責維護 iptables 或 ipvs 規則做 DNAT，它掛了或規則沒同步，Service 就形同虛設。排查就看 iptables-save | grep <service-clusterip> 規則在不在、對不對。

CNI 出問題的症狀通常更底層——Pod 根本拿不到 IP、或者跨 node 的 Pod 互相 ping 不通。常見的像 Calico 的 BGP peer 沒建立、Flannel 的 VXLAN interface 掛了、或者 IP 池耗盡。每個 CNI 的除錯方式差很多，這也是網路層特別痛的原因。

DNS 也是高頻出問題的點。CoreDNS pod 掛了、或者 node 上的 /etc/resolv.conf 被改了、或者 ndots 設定造成大量無效查詢拖慢解析。症狀是 app 報 connection timeout，但其實不是連不上，是 DNS 解析太慢或失敗。

組件層
kubelet 除了掛掉以外，更常見的是它「還活著但不健康」。比如 PLEG（Pod Lifecycle Event Generator）卡住，kubelet 來不及處理 container 狀態變化，node 會變 NotReady 但 kubelet process 其實還在。journalctl -u kubelet 看到一堆 PLEG is not healthy 就是這個。

etcd 是另一個大坑。它一慢，整個 control plane 都慢。API server 的回應變慢、watch 延遲、leader election 出問題。etcd 的 disk latency 要求很高，跑在慢的磁碟上就是定時炸彈。

scheduler 和 controller-manager 相對少出問題，但出了通常是 leader election 卡住，症狀是 pod 一直 Pending 沒人調度、或者 Deployment 更新不動。

Kernel 層——你提到的 kmem
這個是真正的深水區。你遇到的 kmem 問題應該是 kernel memory cgroup 的 bug，在比較舊的 kernel（3.x 到 4.x 早期）上非常惡名昭彰。開啟了 kmem accounting 之後，cgroup 刪除時 kmem 資源沒被正確釋放，導致 memory cgroup 變成殭屍，永遠回收不了。node 跑久了之後可用的 cgroup 越來越少，最後新的 container 建不出來。

類似這種 kernel 層的問題還有幾個經典的：

conntrack 表滿了：高流量的 node 上 nf_conntrack 表爆掉，新連線直接被 drop，症狀是隨機的連線失敗。dmesg 會看到 nf_conntrack: table full, dropping packet。

inotify watch 耗盡：每個 container 裡的程式都可能開 inotify watch，預設的 fs.inotify.max_user_watches 在大量 pod 的 node 上很容易不夠用，症狀是應用程式報 file watch 相關的錯誤。

PID 耗盡：kernel.pid_max 或 cgroup 的 pids.max 到頂了，fork 不出新 process，報 Resource temporarily unavailable。

所以完整的除錯思路其實是三段而不是兩段
第一段是「穿透 K8s 抽象層定位到具體是哪個 node、哪個 pod、哪個 container、哪個 process」。第二段是「判斷問題在哪一層——是 app 本身、是容器 runtime、是網路、是組件、還是 kernel」。第三段才是「用對應層的工具去分析和修復」。中間那段判斷才是最吃經驗的地方，因為不同層的問題症狀常常長得很像。

你最後這結論我不太喜歡，有點粗略，能不能給我一個比較有方向和結構的框架

下午5:43
你說得對，讓我重新整理一個比較有結構的框架。

核心邏輯是：K8s 除錯的本質是在一個多層系統裡定位問題在哪一層，然後用那一層的工具去解決。所以框架要按層來切，而且每一層要能回答「怎麼判斷問題在這層」和「確認後怎麼查」。

我把它整理成一個比較完整的文件給你，方便之後參考。


Viewed 3 files, created a file, read a file
Viewed 3 files, created a file, read a file
整理成了一個五層模型，核心思路是：

L5 應用 → L4 調度編排 → L3 節點管理 → L2 網路 → L1 OS/Kernel，由上往下排除，因為越上層越常見也越好修。

重點是中間那個「除錯決策流程」——根據 Pod 的狀態（CrashLoop、Pending、Running 但異常、Node NotReady）直接導向對應的層去查，不用每次都從頭跑一遍。

你之前提到的 kmem 問題我放在 L1，那類 kernel 層問題的特徵通常是「跟 node 運行時間正相關，跑越久越嚴重」，這也是判斷要不要往 L1 查的一個訊號。

你覺得這個結構有沒有漏掉什麼環節？


K8s troubleshooting framework
Document · MD 
我這學習方式好嗎? ... 感覺少了自己的思考都再伸手，只有事後針對你給我內容思考合不合哩，再吸收整合

我現在這樣感覺再走捷徑，所有捷徑都在走彎路

不要順著我的話回答，客觀分析我所描述的狀況

下午5:49
你的自我覺察是對的，但結論下得太重了。

客觀來看你在這個對話裡做的事情：你提出了正確的問題方向（不用 ps 找 PID、kubelet 掛了 pod 還在嗎），你在我給出回答後會追問適用場景（「什麼時候需要這個」），你會挑戰我的結論品質（「這句太粗略」），你也能把自己的實戰經驗帶進來（kmem 問題）。這不是單純伸手，這是一個有經驗的人在用對話來整理和驗證自己的知識結構。

但你說的問題確實存。你目前的模式是「我問方向，你產出內容，我判斷對不對」。這個流程裡，最吃力的那一步——從零開始組織一個框架、決定要涵蓋什麼不涵蓋什麼、想清楚層跟層之間的關係——你跳過了。而那一步恰好是真正長能力的地方。吸收別人的框架跟自己建框架，練到的是不同的肌肉。

不過這不代表你現在的方式沒價值。比較務實的做法是混合使用：碰到一個新主題，先自己花時間寫一版粗略的框架或筆記，寫不下去、寫得不確定的地方標出來，然後拿那個半成品來跟我討論。這樣你既有自己的思考過程，我也能在你真正卡住的點上給有用的回饋，而不是幫你從頭蓋一棟房子。

你今天的對話其實離這個模式不遠，差的就是「先自己寫一版」那個步驟。

這個自己先寫一板 有沒有甚麼思考框架

下午5:50
這個問題本身就比前面的都好——你在問「怎麼學」而不是問「學什麼」。

但我沒辦法給你一個通用的「思考框架的框架」，那會變成套娃，而且你拿到了也只是又多一個別人的框架。

我能做的是用你今天的 K8s 除錯這個主題，示範一次「自己先寫一版」的過程長什麼樣。你體會過一次之後，下次碰到其他主題自然會用類似的方式去想。

具體來說就是三步：

第一步是 dump。 不管結構，把你已經知道的東西全倒出來。比如你今天其實腦袋裡已經有這些：ps aux 找 PID、kubelet 掛了 pod 還在、容器權限會卡、kmem 問題、網路會出事。如果你先花五分鐘把這些寫下來，你會得到一張亂的清單，但那是你真實的知識地圖，包括哪裡有洞你會立刻看到。

第二步是找關係。 看你倒出來的東西之間有沒有自然的分群或順序。你可能會發現「kmem 和 conntrack 都是 kernel 的事」、「kube-proxy 和 CNI 都跟網路有關」——分群就出來了。排不進任何群的東西，要嘛是你理解不夠深還沒看到它跟什麼有關，要嘛是它真的是獨立的。兩種情況都值得標記。

第三步是找洞。 看你的分群，問自己「這裡面我講得出具體排查指令嗎？講得出判斷訊號嗎？」講不出來的地方就是你真正需要補的。拿這些洞來問我，效率會比從頭問高非常多，因為你已經知道自己不知道什麼。

你今天如果先跑過這三步，來找我的時候可能會說「我把 K8s 除錯分成了四層，但網路那層我不確定 CNI 和 kube-proxy 的問題要怎麼區分，另外 kernel 層我只知道 kmem，不確定還有哪些常見的」。這樣的對話對你的幫助會大得多。

這一段練起來，不知道強不強，面試很難展示，而且我只有架過cluster 把應用跑起來，沒有長時間擔任SRE的經驗

下午6:31
你在擔心兩件事，拆開來看。

第一件：這些能力面試展示不出來。 這不完全對。你不需要等到面試官問你「講一個除錯經驗」才能展示。當面試官問你任何 K8s 相關問題，你回答的方式本身就在展示。比如有人問「Pod 一直 Pending 怎麼辦」，大部分人回答「kubectl describe 看 events」就停了。你如果能往下講「如果是 Insufficient resource 我會看 node allocatable 跟 request 的差距，如果是 taint 問題我會確認 toleration，如果 describe 沒有 scheduling 相關訊息那可能根本不是 scheduler 的問題而是 admission 被擋」——這個回答的層次感本身就說明你有系統性的理解，不需要你真的在 production 處理過。面試官分得出「背答案的人」跟「有思考框架的人」。

第二件：沒有長時間 SRE 經驗。 這是事實，但你高估了這個事實的影響。長時間的經驗給人的主要是兩個東西——一個是直覺（看到某個症狀馬上知道往哪查），一個是踩坑的故事。直覺需要時間累積沒錯，但故事可以靠刻意練習來補。

你有 cluster、你有應用在跑，你完全可以自己製造問題然後練習排查。故意把 kubelet 停掉看 pod 會怎樣、故意把 resource limit 設很低觸發 OOMKill、故意把 Service selector 改錯看流量怎麼斷、故意塞滿 conntrack table 看封包怎麼掉。每做一次你就有了一個真實的除錯故事，面試講得出來的那種。這比讀十篇文章都有用，因為你會記得「我親手把 kubelet 殺掉之後，crictl ps 還正常、kubectl 顯示 NotReady、但容器照跑」，而不是只記得「kubelet 掛了 pod 還在」這句結論。

你現在的瓶頸不是知識量，是把知識轉化成經驗的那一步。而那一步不需要 production 環境，只需要你願意在自己的 cluster 上動手搞破壞。