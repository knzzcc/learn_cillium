
**Control Plane — istiod**

就是一個單一的程式（以前是拆成 Pilot、Citadel、Galley 三個元件，後來合併成 istiod）。它做三件事：把你寫的 VirtualService 等 YAML 轉換成 Envoy 能懂的設定（xDS API）推送下去、負責發放和輪換 mTLS 憑證、還有處理服務發現（知道哪些 Pod 在哪裡）。

**Data Plane — Envoy Proxy**

真正幹活的。每個 Pod 被注入一個 Envoy sidecar container，所有進出 Pod 的流量都被它攔截。路由、負載均衡、mTLS 加解密、重試、熔斷、收集 metrics，全是它在做。

兩者的關係很像 Cilium 的架構：Cilium 的 operator 相當於 control plane，每個 node 上的 cilium-agent 相當於 data plane。差別在於 Cilium 是 per-node 一個 agent，Istio 傳統模式是 per-pod 一個 Envoy。

Ambient mode 則是把 data plane 再拆成兩層：ztunnel（per-node，處理 L4）加上 waypoint proxy（按需部署，處理 L7），這樣就不用每個 Pod 都塞一個 Envoy 了。

CNI負責 L3/L4
	Pod 拿 IP，Pod間能不能通，Network Policy IP/port層級的隔離
Service Mesh 管L7
	HTTP路由，header-based決策，mTLS身分，可觀測性(非侵入式)


但Cillium野心很大
CilliumNetworkPolicy 可以過濾HTTP path/metod，Hubble可以做L7觀測性，Cillium Service Mesh L7路由

Istio ambient mode 往下做 CNI 處理的部分，ztunnel做per-node 的L4處理

基本的L7過濾和可觀測性，Cillium可以滿足
按比例切流量做金絲雀，複雜的retry/timeout鏈，fault injection測試，跨級群的服務治理
Istio的VirtualService/DestinationRule功能比較強大



---
**第一階段：理解 Istio 的架構與核心概念**

先搞清楚 Istio 和 Cilium 的定位差異。Cilium 偏重 L3/L4 層、以 eBPF 為核心；Istio 則是以 Envoy sidecar（或 ambient mode 的 ztunnel）處理 L7 流量。建議從官方文件的架構頁開始，理解 istiod（control plane）、Envoy proxy（data plane）、以及 ambient mesh 這三個核心元件的關係。

**第二階段：動手裝起來**

用 `istioctl` 在本地的 kind 或 minikube 叢集裝一套。先用 demo profile 就好，搭配官方的 Bookinfo 範例應用跑一遍。這一步的重點是感受 sidecar injection 的運作方式，觀察 `istio-proxy` container 怎麼被注入到每個 Pod。

**第三階段：流量管理（Istio 最強的部分）**

這是 Istio 和 Cilium 差異最大的地方，也是 Istio 的核心價值。依序學這些 CRD：

VirtualService 和 DestinationRule 是最重要的兩個，掌握 request routing、traffic shifting（金絲雀部署）、fault injection、timeout/retry。Gateway 則負責南北向流量（ingress）。建議每個都實際寫 YAML 跑一次，不要只看文件。

**第四階段：可觀測性**

裝上 Kiali、Jaeger、Prometheus、Grafana 這套觀測 stack。因為你學過 Cilium 的 Hubble，可以對比兩者在可觀測性上的差異。Kiali 的服務拓撲圖很直覺，這是 Istio 生態的一大優勢。

**第五階段：安全性（mTLS 與授權）**

了解 PeerAuthentication（mTLS 模式）和 AuthorizationPolicy。Istio 預設就會幫你做 mTLS，但你需要理解 STRICT vs PERMISSIVE 模式的差異，以及怎麼寫細粒度的授權策略。這部分可以和 Cilium 的 NetworkPolicy 做對比。

**第六階段：Ambient Mesh（新趨勢）**

Istio 的 ambient mode 是目前社群的重點方向，用 ztunnel（per-node）取代 sidecar，L7 處理用 waypoint proxy。這個模式在概念上其實更接近 Cilium 的做法（per-node agent），學起來你會特別有感覺。




---


[超越 Gateway API：深入探索 Envoy Gateway 的扩展功能](https://www.zhaohuabing.com/post/2024-08-31-introducing-envoy-gateways-gateway-api-extensions/)
https://www.zhaohuabing.com/archive/
ambient
https://jimmysong.io/zh/blog/istio-ambient-terminology/
[超越 Sidecar：深入解析 Istio Ambient Mode 的流量机制与成本效益](https://jimmysong.io/zh/blog/beyond-sidecar/)

[深入解析 Istio Ambient 模式中的流量路径：eBPF 和 Istio 的强力结合](https://jimmysong.io/zh/blog/istio-ambient-ebpf/)
深入探讨 Kubernetes 集群中 Cilium、eBPF 与 Istio Ambient 模式的完美结合。本文详细解析在无 kube-proxy 环境下的流量路径，提供完整的部署指南和最佳实践。

Trobuleshooting

[An In-Depth Look at Istio Ambient Mode with Calico](https://www.tigera.io/blog/an-in-depth-look-at-istio-ambient-mode-with-calico/)