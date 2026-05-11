# [DAY22] Cilium Service Mesh 是什麼？實作 Envoy 流量分流

> 原文連結：https://ithelp.ithome.com.tw/articles/10393923

---

DAY
22

接下來的篇章將踏入 Cilium Service Mesh，後面會想體驗 GatewayAPI 和 Cluster Mesh，但是實際踏進去會發現 Cilium Service Mesh 範疇很大，有一些東西其實我們以前就已經玩到了

這篇我希望先定義好 Cilium Service Mesh 是什麼，然後就會開始從一些好入手的進階功能玩起

先假設我們的系統現在是微服務架構，所以各個微服務都可以獨立部署和運作，各個微服務之間的要溝通就是靠網路在溝通，不像是傳統單體式架構所有的 function call 都在同一個 process 裡面。

如果微服務變得越來越多，在每個 Application 裡面的「對於網路通訊姿勢」都不太一樣就會開始有點麻煩，像是 Service A 和 Service B 的 Retry 邏輯 Timeout 都不太一樣。另一方面就是企業基於安全性會希望微服務之間的通訊要加密，所以會想導入 mTLS

另外我自己比較想玩的其實是金絲雀部署和流量分流，主要是我目前公司在 Disaster Recovery (DR ) 有一些的藍圖想完成以及想提升部署穩定性

我直接舉一個情境好了，來讓讀者感受一下我們有什麼藍圖想達成？現況有遭遇什麼問題：

- 我們服務主要部署在 AWS Singapore Region，如果 AWS 的新加坡機房停電了，部署在新加坡的 Worload 就會直接中斷
- 那假設我們本來就有在 AWS Tokyo Region 做 Active-Active 備援，會希望可以「不改任何 Application 的程式碼」的狀態下：
- Open circuit breaker，所以打去 Singapore 的流量直接被拒絕
- 直接在 Infra 層配置「現在流量 100% 分流去 AWS Tokyo Region」


如果能做到上面這畫面簡直美如畫啊…

這邊是真實的體悟，因為最近 AWS Singapore Region 的 Global Accelerator 其實有發生問題過，導致我們某些 Workload 有中斷…

Cilium 在官方文件有定義 Service Mesh

如果要用一段簡單的話來說，其實 Service Mesh 就是：

👉 **將複雜的網路通訊邏輯從應用程式程式碼中抽離出來，下沉到基礎設施層進行統一管理，而不是散落在應用程式裡 。**

老實說，我覺得在 Cilium 的世界裡，「Service Mesh」這個詞定義有點模糊，而且 Scope 很大。Cilium 已經跨出我原本認為的單純的

Service-to-Service通訊範疇。

看完定義，Service Mesh 就是一種架構、一種設計模式，鐵人賽的學習過程中，其實我們都有在做 Service Mesh 定義中想做的事，因為我們確實有把網路通訊的邏輯抽出來統一管理！最直接的例子是 L7 CiliumNetworkPolicy，我們寫好 L7 CNP，真的做到不用改應用程式，就能控制 client Pod 必須帶特定 Request Header 才可以成功把流量送進 server Pod

如果將 Service Mesh 這種架構，拆成「功能面」來看，一個現代的 Service Mesh 通常會包含這幾個重點：

-
**Resilient Connectivity**：跨 Cluster、跨雲、甚至跨地端的服務通訊，需具備容錯與韌性 -
**L7 Traffic Management**：具備 L7 感知能力的 Load Balancing、Rate Limit、Retry 機制 -
**Identity-based Security**：不再依賴 IP 等網路識別，而是基於 Service Identity 進行雙向驗證 -
**Observability & Tracing**：完整的追蹤與度量能力，用來分析服務穩定度與延遲 -
**Transparency**：所有這些功能對應用程式來說是透明的，不需要修改程式碼

講完了定義，我們就來多玩一些好入手的進階功能，這裡來玩「流量分流」

我們現在要來實踐「**網路通訊邏輯從應用程式程式碼中抽離出來，下沉到基礎設施層進行統一管理」這句話裡面的「基礎設施層進行統一管理」**

至於在 Infra 層為什麼 Pod (也就是我們的 Application) 流量為什麼會在無形之中被 Redirec to Cilium Envoy 請回顧

[Day 17] 為什麼 Cilium 能支援 L7？ Cilium Envoy 解密

我們將用到 EnvoyConfig 來管理流量，但是在這之前會需要安裝 CRD `CiliumEnvoyConfig`

，具體的 Config 是會被寫進去 `CiliumEnvoyConfig`


另外，也可以看一下目前有哪些 Cilium 的 API Resources：

```
$ kubectl api-resources | grep cilium
ciliumcidrgroups ccg cilium.io/v2 false CiliumCIDRGroup
ciliumclusterwidenetworkpolicies ccnp cilium.io/v2 false CiliumClusterwideNetworkPolicy
ciliumendpoints cep,ciliumep cilium.io/v2 true CiliumEndpoint
ciliumidentities ciliumid cilium.io/v2 false CiliumIdentity
ciliuml2announcementpolicies l2announcement cilium.io/v2alpha1 false CiliumL2AnnouncementPolicy
ciliumloadbalancerippools ippools,ippool,lbippool,lbippools cilium.io/v2 false CiliumLoadBalancerIPPool
ciliumnetworkpolicies cnp,ciliumnp cilium.io/v2 true CiliumNetworkPolicy
ciliumnodeconfigs cilium.io/v2 true CiliumNodeConfig
ciliumnodes cn,ciliumn cilium.io/v2 false CiliumNode
ciliumpodippools cpip cilium.io/v2alpha1 false CiliumPodIPPool
```


可以發現 `CiliumEnvoyConfig`

還尚未被註冊進去

現在就來透過 `envoyConfig.enabled=true`

來安裝 `CiliumEnvoyConfig`

CRD，可以透過以下指令進行升級和安裝：

```
$ helm upgrade cilium cilium/cilium --version 1.18.1 \
--namespace kube-system \
--reuse-values \
--set envoyConfig.enabled=true
# 重啟 operator 和 agent
$ kubectl -n kube-system rollout restart deployment/cilium-operator
$ kubectl -n kube-system rollout restart ds/cilium
```


安裝好之後，我們再次檢查看看有沒有 `CiliumEnvoyConfig`

CRD，成功範例如下：

```
$ kubectl api-resources | grep cilium
ciliumcidrgroups ccg cilium.io/v2 false CiliumCIDRGroup
ciliumclusterwideenvoyconfigs ccec cilium.io/v2 false CiliumClusterwideEnvoyConfig
ciliumclusterwidenetworkpolicies ccnp cilium.io/v2 false CiliumClusterwideNetworkPolicy
ciliumendpoints cep,ciliumep cilium.io/v2 true CiliumEndpoint
ciliumenvoyconfigs cec cilium.io/v2 true CiliumEnvoyConfig
ciliumidentities ciliumid cilium.io/v2 false CiliumIdentity
ciliuml2announcementpolicies l2announcement cilium.io/v2alpha1 false CiliumL2AnnouncementPolicy
ciliumloadbalancerippools ippools,ippool,lbippool,lbippools cilium.io/v2 false CiliumLoadBalancerIPPool
ciliumnetworkpolicies cnp,ciliumnp cilium.io/v2 true CiliumNetworkPolicy
ciliumnodeconfigs cilium.io/v2 true CiliumNodeConfig
ciliumnodes cn,ciliumn cilium.io/v2 false CiliumNode
ciliumpodippools cpip cilium.io/v2alpha1 false CiliumPodIPPool
```


接下來，我們透過一個簡單的 canary 流量分流示範來體驗 Cilium Mesh 的運作。

先準備一個 bookstore 範例，包含兩個版本：v1 與 v2。

```
# bookstore-v1-v2.yaml
apiVersion: v1
kind: Service
metadata:
name: bookstore
spec:
selector:
app: bookstore # 注意：Service 同時選擇了 v1 和 v2
ports:
- protocol: TCP
port: 8080
targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
name: bookstore-v1
spec:
replicas: 1
selector:
matchLabels:
app: bookstore
version: v1
template:
metadata:
labels:
app: bookstore
version: v1
spec:
containers:
- name: bookstore
image: busybox:1.36
ports:
- containerPort: 8080
# 使用 command 直接啟動一個極簡 web server
command:
- /bin/sh
- -c
- "while true; do { echo -e 'HTTP/1.1 200 OK\\n\\nResponse from v1'; } | nc -l -p 8080; done"
---
apiVersion: apps/v1
kind: Deployment
metadata:
name: bookstore-v2
spec:
replicas: 1
selector:
matchLabels:
app: bookstore
version: v2
template:
metadata:
labels:
app: bookstore
version: v2
spec:
containers:
- name: bookstore
image: busybox:1.36
ports:
- containerPort: 8080
command:
- /bin/sh
- -c
- "while true; do { echo -e 'HTTP/1.1 200 OK\\n\\nResponse from v2'; } | nc -l -p 8080; done"
```


這裡會建立：

- 2 個 bookstore Deployments，分別是 v1 和 v2
- 1 個
`bookstore`

service，一定要注意這個 Service 直接同時選擇了 v1 和 v2

接著請起一個 `netshoot`

pod，並執行以下指令，這會連續調用 bookstore service：

```
for i in {1..20}; do kubectl exec netshoot -- curl -s http://bookstore:8080; done
```


正常來說執行上面指令後會看到 v1 和 v2 的 Response 是 50:50，因為現在我們還沒有實作流量分流

```
# bookstore-v1-v2.yaml
apiVersion: v1
kind: Service
metadata:
name: bookstore
spec:
selector:
app: bookstore
ports:
- protocol: TCP
port: 8080
targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
name: bookstore-v1
spec:
replicas: 1
selector:
matchLabels:
app: bookstore
version: v1
template:
metadata:
labels:
app: bookstore
version: v1
spec:
containers:
- name: bookstore
image: busybox:1.36
ports:
- containerPort: 8080
command:
- /bin/sh
- -c
- "while true; do { echo -e 'HTTP/1.1 200 OK\\n\\nResponse from v1'; } | nc -l -p 8080; done"
---
apiVersion: apps/v1
kind: Deployment
metadata:
name: bookstore-v2
spec:
replicas: 1
selector:
matchLabels:
app: bookstore
version: v2
template:
metadata:
labels:
app: bookstore
version: v2
spec:
containers:
- name: bookstore
image: busybox:1.36
ports:
- containerPort: 8080
command:
- /bin/sh
- -c
- "while true; do { echo -e 'HTTP/1.1 200 OK\\n\\nResponse from v2'; } | nc -l -p 8080; done"
```


接著我們來真正實作流量分流

原本我們是： 1 個 `bookstore`

service 直接同時選擇了 v1 和 v2

但是我們現在又拆成兩個 service，所以請再幫我 apply 以下配置，這會創建 `bookstore-v1`

和 `bookstore-v2`

Service:

```
# bookstore-svc-v1-v2.yaml
apiVersion: v1
kind: Service
metadata:
name: bookstore-v1
spec:
selector:
app: bookstore
version: v1
ports:
- port: 8080
targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
name: bookstore-v2
spec:
selector:
app: bookstore
version: v2
ports:
- port: 8080
targetPort: 8080
```


創建好 `bookstore-v1`

和 `bookstore-v2`

Services 後，接著需要建立對應的 Envoy 設定，讓流量以 90:10 的比例分配給 v1 / v2，這裡就是會創建 `CiliumEnvoyConfig`

資源：

```
apiVersion: cilium.io/v2
kind: CiliumEnvoyConfig # Cilium 的 CRD，用來下發 Envoy 的 L7 設定
metadata:
name: bookstore-canary
spec:
services:
- name: bookstore
namespace: default # 綁定到 Kubernetes Service (bookstore.default)
backendServices:
- name: bookstore-v1
namespace: default # 後端版本 v1
- name: bookstore-v2
namespace: default # 後端版本 v2
resources: # 定義 Envoy 的完整配置（Listener、Route、Cluster）
# (1) Listener 層：監聽進入 bookstore Service 的 HTTP 流量
- "@type": type.googleapis.com/envoy.config.listener.v3.Listener
name: bookstore-listener
filter_chains:
- filters:
- name: envoy.filters.network.http_connection_manager
typed_config:
"@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
stat_prefix: bookstore-listener
rds:
route_config_name: bookstore-route # 使用下方定義的 route
http_filters:
- name: envoy.filters.http.router # 啟用 HTTP Router 過濾器
typed_config:
"@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
# (2) Route 層：定義流量在 v1 / v2 之間的分配
- "@type": type.googleapis.com/envoy.config.route.v3.RouteConfiguration
name: bookstore-route
virtual_hosts:
- name: bookstore-vh
domains: ["*"] # 匹配所有 domain
routes:
- match:
prefix: "/" # 匹配所有路徑
route:
weighted_clusters: # 加權分流 (Canary Routing)
total_weight: 100
clusters:
- name: "default/bookstore-v1"
weight: 90 # 目前 90% 流量導向 v1
- name: "default/bookstore-v2"
weight: 10 # 10% 比例流量導向 v2，可逐步調整進行 canary rollout
retry_policy: # L7 層級的重試策略
retry_on: 5xx
num_retries: 2
per_try_timeout: 1s
# (3) Cluster 層：定義後端服務連線與健康檢測行為
- "@type": type.googleapis.com/envoy.config.cluster.v3.Cluster
name: "default/bookstore-v1"
connect_timeout: 5s
lb_policy: ROUND_ROBIN
type: EDS # Endpoints Discovery Service，由 Cilium 自動提供 backend endpoints
outlier_detection:
split_external_local_origin_errors: true
consecutive_local_origin_failure: 2 # 若連續 2 次失敗，暫時移除該 endpoint
- "@type": type.googleapis.com/envoy.config.cluster.v3.Cluster
name: "default/bookstore-v2"
connect_timeout: 5s
lb_policy: ROUND_ROBIN
type: EDS
outlier_detection:
split_external_local_origin_errors: true
consecutive_local_origin_failure: 2
```


最後測試看看分流結果：

```
for i in {1..20}; do kubectl exec netshoot -- curl -s http://bookstore:8080; done
```


你應該會看到 v1 和 v2 交錯出現，大部分流量仍打到 v1，但偶爾會有 v2 的回應，這代表 Mesh 層的流量控制生效了。

如果真的不相信，覺得只是運氣好 XD 可以試著把其中一個 service 的比例改成 100，就可以完完全全相信流量分流有奏效

老實說，Service Mesh 一開始看起來真的有點抽象，因為它不是一個單一功能或指令，而是一整套「理念 + 架構 + 工具組合」。

對我來說，最有感的體悟其實是這句話：「**把網路通訊邏輯從應用程式程式碼中抽離出來，交給 Infra 層統一管理。**」

在這篇裡，我們先用比較貼近實務的角度去重新定義 Service Mesh，並從真實的場景出發去看它存在的意義──例如跨 Region 流量分流、災難復原 (DR)、重試與加密通訊等需求。

透過這樣的角度，你會發現 Service Mesh 並不是為了「炫技」或「跟風」，而是當系統規模變大、微服務越來越多時，**這些「非功能性需求」開始變得難以管理，Mesh 的價值就浮現了。**

在技術層面上，我們也實際動手體驗了 Cilium 的一項基礎能力：

透過 `CiliumEnvoyConfig`

將流量分流 (Traffic Splitting) 與重試策略下沉到 Infra 層實作，不改任何應用程式，就能動態控制流量比例。

接下來的篇章會延伸這個基礎，深入探索 **Gateway API** 的實踐

系列文

30 天深入淺出 Cilium ：從入門到實戰
共 30 篇

-
26[Day 26] Cilium Cluster Mesh 實現跨 Cluster 高可用與安全性： Load Balancing 與 NetworkPolicy 實踐
-
27[Day 27] Cilium 實戰分享 (1) 裝了 NodeLocalDNS Cache， DNS 封包原來都沒進去 NodeLocalDNS Cache？
-
28[Day 28] Cilium 實戰分享 (2) 想監控 DNS，封包確實送進去 NodeLocalDNS Cache 了， 但是 hubble_dns_queries_total 怎沒計算到？
-
29[Day 29] Cilium 實戰分享 (3) 裝了 Cilium 後，流量來了，一個 Pod 要 Ready 需要等 26 分鐘？
-
30[Day 30] 深入淺出 Cilium 完結篇：從 Cilium 出發的下一段旅程
