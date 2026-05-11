# [DAY17] 為什麼 Cilium 能支援 L7？ Cilium Envoy 解密

> 原文連結：https://ithelp.ithome.com.tw/articles/10391604

---

DAY
17

在昨天 (Day 16) 的文章，我們已經學會了如何使用 Cilium Network Policy (CNP) 來確保 server Pod 僅允許 client Pod 的 HTTP Request。

原生的 Kubernetes NetworkPolicy 主要在 L3/L4 層級運作，而 Cilium 的 CiliumNetworkPolicy (CNP) / CiliumClusterwideNetworkPolicy (CCNP) 可以指定 HTTP method、path、header 等。

究竟為什麼 Cilium 有辦法做到 L7-aware Network Policy 呢？為什麼 Cilium 有辦法看懂 L7 的資訊？

其實 Cilium 的世界有一個幕後功臣在幫我們處理 L7 的流量 — Cilium Envoy

Cilium Envoy 其實是一個簡化版的 Envoy (去掉不必要的 extension)，並且加上 **Cilium 專用的 policy filter**，用來支援 Cilium 的 L7 Policy。

Cilium Envoy 的 GitHub Repo 在這：github.com/cilium/proxy

**Cilium 有兩種使用 Envoy 的模式**：

-
**內嵌在 cilium-agent**：Envoy 是直接跑在 Cilium Agent 裡面，由 Cilium 自行管理 -
**獨立 DaemonSet**：Envoy 會被獨立部署成一個 DaemonSet，每個 Node 上跑一個 Envoy Pod，和 Cilium Agent 分開

像我的環境是 `external-envoy-proxy: true`

，那 cilium-envoy 會另外自己是一個 DaemonSet，否則他會內嵌在 cilium-agent 內，詳細可以參考官方文件：

```
# 查看 cilium-config 中的 external-envoy-proxy
$ kubectl get cm -n kube-system cilium-config -o yaml | grep external-envoy-proxy
external-envoy-proxy: "true"
# cilium-envoy 單獨 DaemonSet
$ kubectl get ds cilium-envoy -n kube-system
NAME DESIRED CURRENT READY UP-TO-DATE AVAILABLE NODE SELECTOR AGE
cilium-envoy 3 3 3 3 3 kubernetes.io/os=linux 15d
```


至於官方會建議給 Cilium Envoy 獨立的 DaemonSet，主要是有以下優點：

-
**Cilium Agent 重啟不會斷流**Cilium 升級或 Agent 重啟的時候，Envoy 還是繼續跑，不會影響現有的 L7 流量

-
**Envoy 可以獨立升級**要打 Envoy 的 patch 或版本更新，不需要動到 Cilium Agent，兩個分開維護比較乾淨

-
**資源隔離更好**Envoy 跟 Cilium Agent 各自有 CPU/Memory 的 limit，互不干擾，不會搶資源導致流量變卡

-
**Log 分流**Envoy 的 application log 不會跟 Cilium Agent 混在一起，debug 超方便

-
**獨立健康檢查**Envoy 有自己的 health probe，可以單獨監控，出問題也比較好定位

-
**部署行為更明確**DaemonSet 模式就是一開始就直接 deploy 出 Envoy，不像 embedded 模式要等到有 L7 Policy 才動態起來，行為比較 predictable


另外 Cilium Envoy 其實不單單只支援 L7 Network Policy，還支援以下功能：

但今天的篇章著重於 Network Policy 的實現，所以其他功能就不另外介紹

如果讀者已經看過 **[Day 13] 探索 Cilium Pod to Pod Datapath (2) eBPF 走捷徑直接送封包到目的 Pod 面前**，不知道心中會不會有一個直覺：

👉 我們也可以讓封包直接送到 Cilium Envoy 面前

由於 Datapath 的原始碼已經在 Day 12 ~ Day 15 花很大的篇幅講解原始碼。故這裡不會詳細講解原始碼

我們延續昨天 (Day 16) 的情境：

- client Pod@worker-1 向 server Pod@worker-2 發出 HTTP 請求
- server Pod 身上已經有套 CNP 僅允許 client Pod 的 HTTP 請求

而 Datapath 大致上是這樣一路抵達 Cilium Envoy，詳細細節建議讀者回顧 Day 12 ~ Day 15 並點擊我文內提供的原始碼連結很容易就可以追蹤到 :

**1. Egress Policy 檢查點（Client Pod 發送端）**

-
**檔案**：`bpf/bpf_lxc.c:handle_ipv4_from_lxc()`

-
**呼叫鏈**：`policy_can_egress4()`

→`__policy_can_access()`

-
**檢查邏輯**：- 先去
`cilium_policy_v2`

BPF map 撈 policy - 如果有 L3+L4 的精準規則，就比 L4-only 還要優先
- Deny 規則會先跑，中了就直接擋掉；沒中才會去看 Allow
- 如果需要跑 L7，這裡會回傳一個
`proxy_port`


- 先去

-
**L7 Redirect 執行點（Client Pod 發送端）**

-
**檔案**：同樣在`handle_ipv4_from_lxc()`

-
**條件**：`!from_l7lb && proxy_port > 0`

-
**動作**：呼叫`ctx_redirect_to_proxy4()`

-
**結果**：把封包丟去 userspace 的 Envoy Proxy 做 L7 判斷

-
**Ingress 策略決策點 (Server Pod 接收端)**-
**檔案**：`bpf/bpf_lxc.c:ipv4_policy()`

-
**呼叫鏈**：`policy_can_ingress4()`

->`__policy_can_access()`

-
**邏輯**：與 egress 相同，但方向為`CT_INGRESS`


-

看完上面，如果回到我們的情境，封包到底會丟給誰的 Envoy 來做 L7 流量檢查？

是 worker-1 Envoy?

還是 worker-2 Envoy?

老實說，我其實也不知道正確解答，我猜測是是「 L7 Policy 會被套在哪個 Endpoint，就是那 Endpoint 機器上的 Cilium Envoy 來處理」

我們在後面的驗證揭曉！

這裡我們來驗證一下：

- 封包真的會經過 Cilium Envoy
- L7 Network Policy 套在哪個 Endpoint 身上，就是那 Endpoint 機器上的 Cilium Envoy 來處理

目前環境：

-
有三個 Pods，但是待會只會用到 client Pod 和 server Pod

`$ kubectl get po -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES client 1/1 Running 144 (17m ago) 6d2h 10.244.2.173 worker-1 <none> <none> hacker 1/1 Running 25 (11m ago) 25h 10.244.2.57 worker-1 <none> <none> server 1/1 Running 0 2d3h 10.244.1.9 worker-2 <none> <none>`

-
server Pod 身上已經有套 CNP 僅允許 client Pod 的 HTTP 請求

`spec: description: Allow HTTP Request from app=client endpointSelector: matchLabels: app: server ingress: - fromEndpoints: - matchLabels: # 允許 app: client 的 Endpoint Ingress 流量 app: client toPorts: - ports: - port: "80" protocol: TCP rules: http: - headers: - 'X-Secret-Header: secret-value' # 要攜帶此 Request Header method: GET path: /`


我們將使用 `hubble observe`

觀察流量，請多開執行以下命令：

```
# 連進去 worker-1 的 cilium-agent
hubble observe --pod client -f
# 連進去 worker-2 的 cilium-agent
hubble observe --pod client -f
```


接著使用 client Pod 向 server Pod 發出請求：

```
kubectl exec client -- curl -s -H "X-Secret-Header: secret-value" 10.244.1.9
```


worker-1 cilium-agent 實際完整輸出：

```
# worker-1 的 cilium-agent 視角
$ hubble observe --pod client -f
Sep 30 17:06:59.109: default/client:33488 (ID:26328) -> default/server:80 (ID:9764) to-overlay FORWARDED (TCP Flags: SYN)
Sep 30 17:06:59.109: default/client:33488 (ID:26328) <- default/server:80 (ID:9764) to-endpoint FORWARDED (TCP Flags: SYN, ACK)
Sep 30 17:06:59.109: default/client:33488 (ID:26328) -> default/server:80 (ID:9764) to-overlay FORWARDED (TCP Flags: ACK)
Sep 30 17:06:59.109: default/client:33488 (ID:26328) -> default/server:80 (ID:9764) to-overlay FORWARDED (TCP Flags: ACK, PSH)
Sep 30 17:06:59.112: default/client:33488 (ID:26328) <- default/server:80 (ID:9764) to-endpoint FORWARDED (TCP Flags: ACK, PSH)
Sep 30 17:06:59.113: default/client:33488 (ID:26328) -> default/server:80 (ID:9764) to-overlay FORWARDED (TCP Flags: ACK, FIN)
Sep 30 17:06:59.113: default/client:33488 (ID:26328) <- default/server:80 (ID:9764) to-endpoint FORWARDED (TCP Flags: ACK, FIN)
Sep 30 17:06:59.113: default/client:33488 (ID:26328) -> default/server:80 (ID:9764) to-overlay FORWARDED (TCP Flags: ACK)
```


worker-2 cilium-agent 實際完整輸出：

```
# worker-2 的 cilium-agent 視角
$ hubble observe --pod client -f
Sep 30 17:06:59.109: default/client:33488 (ID:26328) -> default/server:80 (ID:9764) policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN)
Sep 30 17:06:59.109: default/client:33488 (ID:26328) -> default/server:80 (ID:9764) to-proxy FORWARDED (TCP Flags: SYN)
Sep 30 17:06:59.109: default/server:80 (ID:9764) <> default/client:33488 (ID:26328) to-overlay FORWARDED (TCP Flags: SYN, ACK)
Sep 30 17:06:59.109: default/client:33488 (ID:26328) -> default/server:80 (ID:9764) to-proxy FORWARDED (TCP Flags: ACK)
Sep 30 17:06:59.109: default/client:33488 (ID:26328) -> default/server:80 (ID:9764) to-proxy FORWARDED (TCP Flags: ACK, PSH)
Sep 30 17:06:59.109: default/server:80 (ID:9764) <> default/client:33488 (ID:26328) to-overlay FORWARDED (TCP Flags: ACK)
Sep 30 17:06:59.111: default/client:33488 (ID:26328) -> default/server:80 (ID:9764) http-request FORWARDED (HTTP/1.1 GET http://10.244.1.9/)
Sep 30 17:06:59.111: default/client:33488 (ID:26328) <> 10.13.1.129 (host) pre-xlate-rev TRACED (TCP)
Sep 30 17:06:59.112: default/client:33488 (ID:26328) <- default/server:80 (ID:9764) http-response FORWARDED (HTTP/1.1 200 1ms (GET http://10.244.1.9/))
Sep 30 17:06:59.112: default/server:80 (ID:9764) <> default/client:33488 (ID:26328) to-overlay FORWARDED (TCP Flags: ACK, PSH)
Sep 30 17:06:59.113: default/client:33488 (ID:26328) -> default/server:80 (ID:9764) to-proxy FORWARDED (TCP Flags: ACK, FIN)
Sep 30 17:06:59.113: default/server:80 (ID:9764) <> default/client:33488 (ID:26328) to-overlay FORWARDED (TCP Flags: ACK, FIN)
```


我們來分析一下 `hubble observe`

的輸出：

-
Policy verdict:

`# worker-2 policy-verdict:L3-L4 INGRESS ALLOWED (TCP Flags: SYN)`

- 在 server Pod (worker-2) 的 ingress pipeline 上，先做了 L3/L4 policy 判斷
- verdict =
**ALLOWED**，表示基本的 L4 規則（TCP/80 from client）過關

-
to-proxy

`# worker-2 -> default/server:80 (ID:9764) to-proxy FORWARDED (TCP Flags: SYN) -> default/server:80 (ID:9764) to-proxy FORWARDED (TCP Flags: ACK) -> default/server:80 (ID:9764) to-proxy FORWARDED (TCP Flags: ACK, PSH)`

-
**關鍵證據**：這些 event 說明流量被**redirect 到 proxy (cilium-envoy)** - 只有命中 L7 規則才會看到
`to-proxy`

- 如果只有 L3/L4，這裡應該是
`to-endpoint`

，不會有`to-proxy`


-
-
http-request / http-response

`# worker-2 http-request FORWARDED (HTTP/1.1 GET http://10.244.1.9/) http-response FORWARDED (HTTP/1.1 200 0ms (GET http://10.244.1.9/))`

- 這表示 Envoy 已經接收到流量並解碼 HTTP
- 能看到 method = GET, path =
`/`

，status = 200，這只有 Envoy L7 proxy 解析才可能出現 - 也證明 CNP 的 L7 規則（GET / with header）確實在運作

-
worker-1 的輸出沒看到任何

`to-proxy`

- 所以這意味著我們沒用到 worker-1 身上的


所以我們可以得出結論：

✅ 流量確實有經過 **cilium-envoy**：`to-proxy`

和 `http-request/response`

就是鐵證

✅ L7 Network Policy 套在哪個 Endpoint 身上，就是那 Endpoint 機器上的 Cilium Envoy 來處理

昨天 (Day 16) 我們已經用 Cilium Network Policy (CNP) 實驗過 server Pod 只能收 client Pod 的 HTTP Request。這裡我們更進一步驗證了流量真的會被導到 **Cilium Envoy**，並且讓它來做 L7 的判斷。

從 `hubble observe`

的輸出可以很明顯看出幾個關鍵點：

-
**policy verdict**先跑 L3/L4 的 policy 檢查，確認 TCP/80 from client 是 OK 的。

-
**to-proxy**這是最關鍵的證據，代表封包沒有直接送進 server Pod，而是被 redirect 到本機的

`cilium-envoy`

。如果只是 L3/L4 policy，這裡會看到`to-endpoint`

而不是`to-proxy`

。 -
**http-request / http-response**Envoy 幫忙解碼 HTTP，顯示出 method、path、header、status code 等等。這也就證明了為什麼 Cilium 有能力去 enforce L7-aware 的 policy ， 因為背後其實就是靠 Envoy 做解析。

-
L7 Network Policy 套在哪個 Endpoint 身上，就是那 Endpoint 機器上的 Cilium Envoy 來處理


所以整個流程就是：

client Pod 發出 HTTP → 先過 L3/L4 policy → 被 redirect 到 Cilium Envoy → Envoy 解析 HTTP header/path/method → 最後再決定要不要放行

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
