# [DAY01] 前言 & 為什麼要用 Cilium？要解決什麼痛點？

> 原文連結：https://ithelp.ithome.com.tw/articles/10379918

---

DAY
1

Shiun 目前於 SHOPLINE 擔任 Cloud Engineer，工作內容主要專注在 EKS、Networking 與 Infra，平時就很喜歡鑽研 Serverless, Container, Networking 等技術的底層原理

同時也是 AWS Community Builder 以及 AWS Educate Cloud Ambassador，積極參與社群活動與推廣雲端教育

個人部落格：https://shiun.me

源自於自己在學習 Cilium 時，發現中文圈還沒有「Cilium 的完整教學文章」（也許有但可能我還沒發現 ><）， 也發現中文圈的 Cilium 學習資源都還很分散，應該很多正在學 Cilium 的台灣人都有共鳴。~~當然，對於英文閱讀或是聽力能力不錯的人這方面絕對是沒困擾的~~

加上 Cilium 的學習曲線很高，主要是因為它會把你帶進去 Linux kernel 與 eBPF 的世界，涉及的技術層面又廣又底層，要完全掌握住真的要花蠻多時間去研究和踩坑的。

而當真的下手去研究，踏進去 Cilium 的官方文件也容易迷路在裡面，因為資訊量很大很廣，常常不知道該從哪邊下手

所以我才想在中文圈寫一篇 Cilium 入門到實戰的系列文，希望能幫助到跟我一樣想學習 Cilium 的人！

**有 K8s 使用經驗以及 Networking 基本知識**，但對 Cilium 與 eBPF 的內部原理還很陌生，並且想要開始使用或是剛開始使用 Cilium 的人。

首先，我們要先理解 K8s 要解決最基本的 Pod 網路通訊問題 (例如：建立 veth pair, assign IP address 等等)，背後是靠一個標準化的介面 (或是說規範) - CNI (Container Network Interface) 來負責，依照此規範實作出來的東西就是所謂的 **CNI Plugin** 。像 Calico、Flannel、Weave Net 都是常見的 CNI Plugin

另外就是 Kubernetes Service，一個 Service 會提供一個虛擬 IP (ClusterIP) 和 DNS Name 作為一群 Pods 的統一入口，而背後能把 ClusterIP 「翻譯」成實際的 Pod IP 的功臣就是運行在每個節點上的 `kube-proxy`

，它會監聽 API Server 的變化，一旦有 Service 或其後端 Pod（Endpoints）發生變動，它就會在節點上設定對應的 iptables rules (這裡不討論 IPVS)

以下就是一個 K8s Service，其背後的 iptables rules 範例：

```
$ iptables-save | grep "KUBE-SVC-N3PJZKC3AAWBF4KR" | grep "shiun"
-A KUBE-SERVICES -d 172.20.235.205/32 -p tcp -m comment --comment "shiun/shiun-nginx:http cluster IP" -m tcp --dport 80 -j KUBE-SVC-N3PJZKC3AAWBF4KR
-A KUBE-SVC-N3PJZKC3AAWBF4KR -m comment --comment "shiun/shiun-nginx:http -> 10.13.112.110:8080" -m statistic --mode random --probability 0.33333333349 -j KUBE-SEP-QYTNDCGG4QDC4YF4
-A KUBE-SVC-N3PJZKC3AAWBF4KR -m comment --comment "shiun/shiun-nginx:http -> 10.13.4.117:8080" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-U6AZMN52AX32BYAE
-A KUBE-SVC-N3PJZKC3AAWBF4KR -m comment --comment "shiun/shiun-nginx:http -> 10.13.95.137:8080" -j KUBE-SEP-4WPJATPLDHWMRVLT
```


以上提到的 CNI Plugin 及 `kube-proxy`

，都仰賴 **iptables** 來實現 Network Policy 及 Service Load Balancing，這就帶來幾個問題：

-
**效能瓶頸**：iptables 是「一條一條規則順序比對」，時間複雜度是**O(n)，**規則數量一多，效能就會明顯下降 -
**可觀測性不足**：你很難清楚看到「是哪個 Pod 封包被丟掉」或「哪條規則生效」 -
**管理複雜**：IP 綁定 Policy，Pod 一旦 Churn (動態變動很快)，會導致大量規則需要更新

這時候，eBPF (extended Berkeley Packet Filter) 出場了，eBPF 的細節在之後幾天會更深入探討，這裡先簡單來說明 eBPF 是什麼。eBPF 就是「**在不用重新編譯 Linux Kernel 或是修改 Linux Kernel 程式碼的情況下，安全地掛載一段自訂程式碼到 Kernel 中的特定點 (hook points)**」


圖片取自：https://ebpf.io/what-is-ebpf/#hook-overview

eBPF 的感覺大概是這樣：

- 傳統方式：所有網路流量都得經過「固定管線」，如果想改東西就要改 Kernel 原始碼然後重新編譯
- eBPF 方式：在 hook point 掛上 BPF 程式，彈性攔截封包、做決策，甚至輸出監控資料

那這有什麼好處？

-
**更快**：不用跑一大堆 iptables 規則，直接在 Kernel 裡處理 -
**更聰明**：可以看到每個封包的 Metadata，做更細的判斷，甚至直接修改封包 -
**更可觀測**：透過 eBPF，把封包資訊即時送到 Hubble 這樣的工具，即時看到誰打誰、什麼流量被 Drop

Cilium 就是基於 eBPF 打造的 **新一代 CNI Plugin**，它能：

-
**取代 kube-proxy**：用 eBPF 加速 Service Load Balancing -
**Identity-based Policy**：不靠 IP，而是靠 Pod Label 來分配 Identity ，並基於 Identity 來決策流量 -
**內建 Observability**：Hubble 提供即時流量可視化

基於以上，Cilium 能夠**真正解決前面提到的痛點**，它讓 Network Policy 更穩定、效能更好、可觀測性更強

所以接下來就跟著我解鎖 Cilium 、掌握 Cilium！


