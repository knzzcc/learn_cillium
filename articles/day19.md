# [DAY19] Cilium 如何在沒有 kube-proxy 的情況下實現 K8s Service？

> 原文連結：https://ithelp.ithome.com.tw/articles/10392583

---

DAY
19

看完了 Pod-to-Pod，以及 Pod 通信之間的安全性該如何落實後，讓我們放大層級，來思考 Kubernetes Service 層級的事情。

這裡就要講到 Cilium 很具代表性的特色 — **kubeProxyReplacement**

這功能在我們 Day 2 安裝 Cilium 時就有開啟：

```
helm install cilium cilium/cilium \
--set kubeProxyReplacement=true
```


那 Cilium 是怎麼把 kube-proxy 擠下舞台，是如何取代的呢？背後是有什麼奧秘才能讓我們 Kubernetes Service 運作的又快又穩呢？

推薦影片：

Liberating Kubernetes From Kube-proxy and Iptables - Martynas Pumputis, Cilium

*圖片取自 K8s 官方：https://kubernetes.io/docs/reference/networking/virtual-ips/*

Kubernetes 引入了 **Service** 這個核心抽象 。

Service 擁有一個穩定的 Virtual IP — 稱為 `ClusterIP`

， Client 端可以持續地與這個 `ClusterIP`

通信，而無需關心後端 Pod 的變化 。

而 kube-proxy 正是 Kubernetes 的 **Service 實現層**，它負責把「抽象的 Service」轉換成「實際的封包轉送規則」，預設是透過 iptables 來實現封包轉送規則，但是 iptables 有一大缺點是「一條一條規則比對」，所以在查找規則時是 **O(n)**

另外 kube-proxy 除了 iptables 模式之外，還可以使用 **IPVS 模式**，工作方式會變成：

- 為每個 Service 創建一個 IPVS Virtual Server
- 為每個後端 Endpoint 創建一個 IPVS Real Server，並將其關聯到對應的 Virtual Server

IPVS 與 iptables 相比，性能會比較好，因為它底層資料結構使用 hash table 來儲存 Service 與 Backend 的映射關係，所以查找操作的時間複雜度達到為 **O(1)**

當然我知道有個繼承者 — nftables，但這裡先不討論


換句話說：

👉 kube-proxy 是讓你用 **Service ClusterIP / NodePort / LoadBalancer IP** 能夠順利打到後端 Pod 的關鍵元件

老實說 kube-proxy 不是不能使用，大多數中小型 cluster 使用 kube-proxy 也能運作得好好的。

那為什麼會有人選擇「不使用 kube-proxy」呢？主要原因有以下：

-
**效能瓶頸**- kube-proxy 在
**iptables 模式**下，是靠一條一條規則去匹配封包 - Service / Endpoint 一多（成千上萬 Pod），iptables 會變成
**O(n) 查找**，效能直線下滑 - 規則更新也很慢，因為改動要重新灌整個 ruleset
- 後來 kube-proxy 支援
**IPVS**，效能比 iptables 好很多（hash-based 查找，O(1)）。 - 但 IPVS 本質上還是額外一層封包處理，規則同步依賴 kube-proxy，不算根本解決方案

- kube-proxy 在
-
**可觀測性不足**- 想 trace 一條連線經過哪個 Service → Pod？基本上只能靠 conntrack / tcpdump 苦工
- 沒有 built-in observability，
~~用過 hubble 的都回不去了吧…~~


所以以 eBPF 為基底的 Cilium 直接把 Service LB 拉進 eBPF Datapath 實作出來，解決了上述的痛點，所以我們使用 Cilium 才有辦法跟 kube-proxy 說掰掰。

再加上 Hubble 提供的可觀測性，Cluster 的封包流動可以被輕鬆掌握清楚。

相信已經看過前面本鐵人賽系列文前幾篇文章的讀者都知道 eBPF 有各種走捷徑的方式了


上面光講實在不夠，以下我附上 Isovalent 官方實測的 eBPF vs IPVS vs iptables 效能表現：

*資料來源：https://isovalent.com/blog/post/why-replace-iptables-with-ebpf/*

上圖中的 `TCP_CRR`

是 **TCP Connect Request/Response** 的縮寫，其概念是：每個 request 都會重新建立 TCP 連線（三次握手）→ 傳送 request/response → 然後關閉（四次揮手）。

會發現：**當 service 數量增加時，eBPF 幾乎維持穩定 latency**，所以**大規模 cluster 會更凸顯 eBPF 的優勢**。

這裡我們先來看概覽就好，另外想提一下， `kubeProxyReplacement`

的 Pod-to-Service Datapath 真的又龐大又複雜… 我實際去追原始碼一探究竟封包是怎麼被 eBPF Program 左右命運時，覺得**非常可怕** XD （很複雜

先假設現在環境現在長這樣：

```
ClusterIP: 10.96.0.1:80
Backends: 10.244.1.10:80, 10.244.2.15:80
```


→ 如果 Pod 去打 `10.96.0.1:80`

，我們預期流量會送到 Backends 上的某個 Pod

Cilium 完全不用 iptables 或 IPVS，而是靠 eBPF + BPF Maps。

-
**攔截點：connect() syscall**- 當 Pod 內應用程式呼叫
`connect(ClusterIP:Port)`

時， - eBPF 程序在
**cgroup/inet4_connect hook**攔截。

- 當 Pod 內應用程式呼叫
-
**查 Service Map**- 以
`(ClusterIP, Port, Proto)`

當 key， - 從
`cilium_lb4_services_v2`

查找對應 Service。

- 以
-
**選擇 Backend**- 支援 RR、Maglev、一致性哈希、Session Affinity。
- 查
`cilium_lb4_backends_v3`

得到 backend Pod`IP:Port`

。

-
**改寫 socket 參數**-
在

`cil_sock4_connect()`

中：`ctx->user_ip4 = backend->address; ctx_set_port(ctx, backend->port);`

-
意思是

**直接改掉 struct sockaddr 的目標**。

-
-
**建立 Socket → 產生封包**- TCP SYN 封包出來時，
**dstIP 已經是 Pod IP**。 - 因此無論在 Pod veth、host veth、VXLAN 上都看不到 Service IP。

- TCP SYN 封包出來時，
-
**回程流量**- 因為 socket 已直接連到 Pod，不需要 DNAT / checksum 修正。
- Service IP 完全不會出現在任何封包裡。


今天的文章比較算是點出一個概觀的概念，以下我整理出了 iptables vs IPVS vs Cilium 的表格：

| 功能 | iptables | IPVS | Cilium (eBPF) |
|---|---|---|---|
| 規則查找 | O(n)，線性掃描 | O(1) hash table | O(1) BPF map |
| 規則更新 | 重寫整張表，秒級中斷 | 增量更新 | BPF map 原子更新 |
| NAT | iptables DNAT | IPVS NAT | cgroup hook 攔截 |
| conntrack | kernel conntrack | kernel conntrack | eBPF CT map |
| 擴展性 | 差 | 中等 | 高，支援 L7 / Maglev / Session Affinity |

**一句話總結今天文章**：

Cilium 透過 eBPF 在 Kernel datapath 中實作 **Socket LB**，利用 BPF Maps 進行 O(1) 的 Service/Backend 查詢，並內建 Connection Tracking 機制，從而徹底取代 kube-proxy。
