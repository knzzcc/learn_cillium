常見容器化環境debug工具 包含k8s/docker不限於網路層面

下午1:57

2 / 2

## Claude responded: 常見容器化環境 Debug 工具

## 常見容器化環境 Debug 工具

以下整理 Docker 與 Kubernetes 環境中常用的除錯工具，涵蓋網路、日誌、效能、儲存等多個面向。

------

### Docker 層級

**內建指令**

Docker 本身就提供不少實用的除錯手段。`docker logs` 搭配 `--follow` 和 `--since` 可以即時追蹤或回溯容器日誌。`docker exec -it <container> sh` 直接進入容器互動式 shell，是最直覺的做法。`docker inspect` 能查看容器的完整設定，包括網路、掛載、環境變數等 metadata。`docker stats` 則提供即時的 CPU、記憶體、網路 I/O 概覽，類似簡易版的 top。`docker diff` 可以看出容器檔案系統相對於映像檔有哪些變更（新增、修改、刪除），對追查「誰動了什麼檔案」很有用。`docker events` 會即時串流 daemon 層級的事件，例如容器啟動、停止、OOM kill 等。

**網路除錯**

容器內如果沒有預裝工具，可以用 `docker run --net=container:<target> nicolaka/netshoot` 把 netshoot 這個工具箱掛到目標容器的 network namespace 裡，裡面包含 tcpdump、nslookup、curl、iftop、iperf 等常見網路工具。另外 `docker network inspect` 可以查看特定 bridge/overlay 網路上有哪些容器、各自拿到什麼 IP。

**映像檔分析**

`dive` 是一個很受歡迎的開源工具，能逐層檢視 Docker image 的每一個 layer 裡面加了哪些檔案、各佔多少空間，對於瘦身映像檔和排查「為什麼 image 這麼大」非常好用。`docker history <image>` 也能快速看到各層的指令和大小，但沒有 dive 那麼直觀。

------

### Kubernetes 層級

**內建指令**

`kubectl describe pod/node/svc` 是第一步，Events 區段通常就能看出排程失敗、拉取映像失敗、探針失敗等問題。`kubectl logs` 支援 `--previous`（看前一次 crash 的日誌）和 `-c`（指定 init container 或 sidecar）。`kubectl get events --sort-by=.lastTimestamp` 可以看到叢集層級的事件時間軸。`kubectl top pod/node` 需要 metrics-server，能看即時資源用量。

**kubectl debug**

從 Kubernetes 1.23 GA 的 ephemeral container 功能，`kubectl debug -it <pod> --image=busybox --target=<container>` 可以在不重啟 pod 的前提下注入一個臨時容器，共享目標容器的 PID/network namespace，適合 distroless 映像或 production 環境不方便裝工具的場景。也可以用 `kubectl debug node/<node-name> -it --image=ubuntu` 直接 debug 節點。

**k9s**

終端機上的互動式 Kubernetes dashboard，能快速切換 namespace、查看 pod 狀態、即時看日誌、進入 shell、刪除資源等，比反覆打 kubectl 指令快很多。支援自訂快捷鍵和 plugin。

**Lens / OpenLens**

圖形化的 Kubernetes IDE，適合偏好 GUI 的人。可以管理多個叢集、查看資源拓撲、即時日誌和 shell，內建 Prometheus 指標整合。

------

### 網路專項

`tcpdump` 和 `Wireshark` 是抓封包的經典組合，在容器內跑 tcpdump 抓 pcap 檔再用 Wireshark 開啟分析。`ksniff` 是一個 kubectl plugin，可以一鍵對指定 pod 啟動 tcpdump 並自動導入本地 Wireshark，省去手動操作。

`cilium` 和 `hubble` 如果叢集用的是 Cilium CNI，Hubble 提供基於 eBPF 的服務間流量可觀測性，包括 L3/L4/L7 的 flow log、DNS 查詢、HTTP 狀態碼分布等，而且不需要 sidecar。

對於 service mesh 環境（Istio/Linkerd），`istioctl analyze` 可以檢查設定錯誤，`istioctl proxy-status` 和 `proxy-config` 能查看 Envoy sidecar 的同步狀態與路由規則。

------

### 日誌與可觀測性

EFK/ELK stack（Elasticsearch + Fluentd/Filebeat + Kibana）或 Loki + Grafana 是常見的集中式日誌方案。Loki 的優勢在於不做全文索引、只索引 label，資源消耗低得多。`stern` 是一個小工具，可以同時 tail 多個 pod 的日誌並以顏色區分，比原生 kubectl logs 方便很多。

監控面則是 Prometheus + Grafana 的組合最主流，搭配 Alertmanager 做告警。分散式追蹤可以用 Jaeger 或 Zipkin，搭配 OpenTelemetry SDK 埋點，排查跨服務的延遲瓶頸。

------

### 效能與資源

`cAdvisor` 由 Google 開發，能收集每個容器的 CPU、記憶體、檔案系統、網路等指標，kubelet 內建就有。`perf`、`strace`、`bpftrace` 這些 Linux 原生工具在容器內也能用（需要適當的 capability），適合深入排查系統呼叫層級的效能問題。`kubectl-flame` 是一個 kubectl plugin，能對運行中的 pod 做 CPU profiling 並產生火焰圖，支援 Java、Go、Python、Node.js 等語言。

------

### 儲存與設定

PV/PVC 問題可以先用 `kubectl describe pv/pvc` 看 Events，常見的是 StorageClass 不存在、權限不足、或節點 affinity 不符。如果需要檢查 PV 的實際內容，可以起一個臨時 pod 掛載該 PVC 來查看。`kubectl get configmap/secret -o yaml` 搭配 `base64 -d` 可以檢查注入到 pod 的設定是否正確。

------

### 安全與政策

`kubectl auth can-i` 可以快速確認某個 ServiceAccount 或 User 是否有特定權限。`kube-bench` 檢查叢集是否符合 CIS Kubernetes Benchmark。`trivy` 可以掃描容器映像的 CVE 漏洞，也能掃描 Kubernetes YAML 的設定風險。`falco` 則是執行期的安全監控，基於 eBPF/kernel module 偵測異常行為（例如容器內執行了 shell、讀取了 /etc/shadow）。

------

簡單的優先順序建議：遇到問題先從 `describe` + `logs` + `events` 看起，大多數情況答案就在裡面。需要深入網路層就上 netshoot 或 tcpdump，效能問題看 metrics 和 profiling，複雜的微服務問題則靠集中式日誌和分散式追蹤來定位。

## You said: 那ss這些呢?

那ss這些呢? dmesg

下午1:59

## Claude responded: 好問題，這些是 Linux 系統層級的工具，在容器環境中同樣非常實用，只是使用上有一些 namespace 和權限的眉角。

好問題，這些是 Linux 系統層級的工具，在容器環境中同樣非常實用，只是使用上有一些 namespace 和權限的眉角。

------

### 網路狀態類

**ss（Socket Statistics）**

取代 netstat 的現代工具，速度更快（直接讀 kernel 的 netlink socket，不走 /proc 解析）。在容器內跑 `ss -tlnp` 可以看該容器 network namespace 內正在 listen 的 port 和對應 process，`ss -s` 看 socket 統計摘要（有多少 TCP established、TIME-WAIT 等），`ss -ti` 則能顯示每條連線的 TCP 內部資訊，包括 cwnd、rtt、retrans 等，對排查連線品質和壅塞問題很有用。常見的除錯場景是用 `ss -tnp state time-wait | wc -l` 確認是否有 TIME-WAIT 堆積的問題。

**ip（iproute2 套件）**

`ip addr` 看介面和 IP、`ip route` 看路由表、`ip neigh` 看 ARP 表、`ip link` 看介面狀態。在容器內跑這些可以確認 veth pair 的狀態、預設路由是否正確指向 bridge gateway 等。搭配 `ip netns` 在宿主機上可以列出和進入各個 network namespace，直接檢查特定容器的網路視角。

**nstat / netstat**

`nstat` 可以看 SNMP 計數器的變化量，比如 TCP retransmit、UDP overflow 等，對排查丟包和重傳很有幫助。netstat 雖然被視為 legacy 但有些 distroless 環境反而會透過 netshoot 帶進來用。

------

### 系統日誌類

**dmesg**

讀取 kernel ring buffer 的訊息，在容器環境中特別重要的場景包括 OOM kill（`dmesg | grep -i oom` 可以看到哪個 cgroup 的哪個 process 被殺、當時的記憶體狀況）、磁碟 I/O 錯誤、網路介面異常、以及 iptables/nftables 的 LOG target 輸出。需要注意的是，容器內預設通常沒有權限執行 dmesg（因為 `dmesg_restrict` sysctl），需要 `CAP_SYSLOG` 或 `--privileged`，所以實務上更常在宿主機或透過 `kubectl debug node/` 來執行。

**journalctl**

如果宿主機用 systemd，`journalctl -u kubelet` 查看 kubelet 的日誌、`journalctl -u containerd` 或 `journalctl -u docker` 查看 container runtime 日誌。搭配 `-f`（follow）和 `--since "5 minutes ago"` 可以快速縮小範圍。kubelet 層級的錯誤（例如 CNI plugin 失敗、volume mount 失敗）在 kubectl 上不一定看得到，得從這裡找。

------

### 效能與行為追蹤類

**strace**

追蹤 process 的 system call，`strace -p <pid> -e trace=network` 只看網路相關的 syscall，`-e trace=file` 只看檔案操作。排查「為什麼 process 啟動後卡住」非常有效，通常能看到是卡在 DNS 查詢、連線某個 endpoint、或讀取某個 config 檔。在容器內需要 `CAP_SYS_PTRACE`，或者用 ephemeral container 搭配 `shareProcessNamespace: true` 來做。

**lsof**

`lsof -i :8080` 看誰佔了某個 port，`lsof -p <pid>` 看某個 process 開了哪些 file descriptor（包括 socket、pipe、regular file）。排查「為什麼 port 衝突」或「file descriptor 洩漏」的經典工具。

**/proc 檔案系統**

容器內也可以直接讀 /proc 來獲取豐富資訊。`/proc/net/tcp` 和 `/proc/net/tcp6` 是 ss 的底層資料來源；`/proc/<pid>/status` 看 process 的記憶體用量和狀態；`/proc/<pid>/limits` 看 ulimit 設定（常見的坑是 nofile 太低導致連線數不夠）；`/proc/<pid>/fd/` 列出所有打開的 file descriptor。

------

### 容器環境的權限限制

這些工具在容器中最常遇到的障礙就是 Linux capabilities 不足。整理一下對應關係：

| 工具                 | 需要的權限       | 取得方式                                   |
| -------------------- | ---------------- | ------------------------------------------ |
| ss、ip               | 通常不需額外權限 | 預設可用                                   |
| dmesg                | CAP_SYSLOG       | securityContext 或 privileged              |
| strace               | CAP_SYS_PTRACE   | securityContext 加上 shareProcessNamespace |
| tcpdump              | CAP_NET_RAW      | securityContext 或用 netshoot              |
| iptables / nftables  | CAP_NET_ADMIN    | securityContext                            |
| mount（檢查 volume） | CAP_SYS_ADMIN    | 通常只在 debug 用 privileged pod           |

實務上建議的做法是不要為了除錯把 production pod 改成 privileged，而是用 `kubectl debug` 的 ephemeral container 或另起一個有權限的 debug pod 掛入同一個 namespace 來操作。這樣既能用到完整工具，又不會影響正式服務的安全邊界。

## You said: 幫我編一套，從k8s cri架構到權限和debug工具

幫我編一套，從k8s cri架構到權限和debug工具

下午2:14

## Claude responded: Let me read the relevant skill files first.

Let me read the relevant skill files first.

Viewed 2 files

I'll create this as an interactive React reference guide covering the full stack from CRI architecture to debug tooling.

Created a file, read a file

整理了一份七個分頁的互動式參考手冊，結構從底層到實戰：

1. **CRI 架構** — kubelet → CRI gRPC → containerd/CRI-O → runc → kernel 的完整呼叫鏈，以及 crictl、ctr 等 CRI 層 debug 指令
2. **Namespace 隔離** — 七種 Linux namespace 的作用與 debug 意義，nsenter 用法，cgroup 檢查
3. **權限模型** — Capabilities、Seccomp、AppArmor 三道防線，SecurityContext 設定範例，以及每個 debug 工具需要什麼 cap 的速查表
4. **網路 Debug** — ss、ip、tcpdump、nstat、DNS 測試、Cilium/Hubble 全收錄
5. **系統 Debug** — dmesg、strace、lsof、/proc 檔案系統、perf/bpftrace、journalctl
6. **可觀測性** — 日誌（EFK/Loki/stern）、指標（Prometheus）、追蹤（Jaeger/OTel）、安全（Falco/Trivy）
7. **Playbook** — CrashLoopBackOff、Service 連不到、應用變慢、Volume 掛載失敗四個常見場景的 step-by-step 排查流程

每個工具都附帶具體指令和使用場景說明，可以直接當 cheat sheet 用。





K8s debug reference

Code · HTML 

Download

## You said: 這一份run過 會很扎實吧?

這一份run過 會很扎實吧?

下午2:57

## Claude responded: 會，這份內容量蠻紮實的。從架構原理到實戰指令都有涵蓋，簡單盤點一下：

會，這份內容量蠻紮實的。從架構原理到實戰指令都有涵蓋，簡單盤點一下：

**知識面**涵蓋了 CRI 呼叫鏈（kubelet → containerd → runc → kernel）、七種 Linux namespace 的隔離機制、Capabilities/Seccomp/AppArmor 三層權限模型，這些是很多人用了幾年 K8s 也不一定能完整說清楚的底層知識。

**工具面**大概收了 30+ 個工具和指令，從不需權限的 ss、ip、/proc，到需要特定 cap 的 strace、tcpdump、dmesg、bpftrace 都有，而且每個都標了需要什麼權限、建議怎麼取得，這在實際操作時很關鍵——不然常常卡在 "permission denied" 不知道要加什麼。

**Playbook 部分**是最實用的，四個場景（CrashLoopBackOff、連線問題、效能下降、Volume 掛載失敗）基本上覆蓋了日常 on-call 最常遇到的狀況，每個都是一步步排除的邏輯鏈。

如果要再補強，可以考慮加幾個方向：

- **Ingress / Service Mesh 層的除錯**（Istio proxy-config、envoy access log）
- **etcd 層面的問題排查**（leader election、compaction、DB size）
- **節點層級的問題**（NotReady 排查、kubelet 啟動失敗、CNI plugin 故障）
- **實際 lab 練習**，比如故意製造 OOM、NetworkPolicy block、seccomp 違規等讓人動手排

要不要我把這些補進去，或是另外做一份進階版？