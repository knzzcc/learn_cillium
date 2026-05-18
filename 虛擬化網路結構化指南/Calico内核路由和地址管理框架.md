calico 內核當router，地址浪費 arp 一步一步來 消化不來 給我一個結構化的框架，理解calico

5月15日

## Claude responded: 我來幫你建立一個結構化的框架來理解 Calico。我們一步一步來，從最核心的概念開始。

我來幫你建立一個結構化的框架來理解 Calico。我們一步一步來，從最核心的概念開始。

------

Calico 的核心思想可以用一句話概括：**把每個 Pod 當成互聯網上的一台主機，用標準的 IP 路由（Layer 3）來轉發封包，而不是用 overlay 網路的封裝。**

理解 Calico 可以分成這幾個層次，我建議你按這個順序消化：

------

**第一層：為什麼要用 L3 路由？（動機）**

傳統的 K8s 網路方案（如 Flannel VXLAN）會把 Pod 的封包用 VXLAN 包一層再送出去，這是 overlay。Calico 的做法不同——它說每個 Node 就是一台 router，Pod 的流量直接靠路由表轉發，不封裝。好處是效能好、可見性高（tcpdump 直接看到原始封包）、也能直接跟外部 BGP 路由器對接。

------

**第二層：Node 內部怎麼運作？（單機模型）**

這是你提到「內核當 router」的部分。每個 Node 上發生的事：

每個 Pod 會拿到一個 veth pair，一端在 Pod 的 network namespace 裡，另一端在 host 上。關鍵在於 Calico **不用 bridge**，而是讓 host 端的 veth 直接當作一個 point-to-point 的路由介面。Linux kernel 的路由表裡會寫一條路由，例如 `10.233.1.5 dev caliXXXX`，意思是「要到這個 Pod IP，就從這個 veth 介面送出去」。所以 Linux kernel 本身就是 router，每個 veth 就是一個 next-hop 介面。

------

**第三層：你問的 ARP 問題**

因為 Calico 用的是 point-to-point 的 L3 模型（想像成每條 veth 是一條點對點鏈路），Pod 裡的 default gateway 被設成 `169.254.1.1`（一個 link-local 地址）。Pod 要送封包出去，需要 ARP 解析這個 gateway 的 MAC。

Calico 的做法是在 host 端的 veth 上開啟 **proxy ARP**（`/proc/sys/net/ipv4/conf/caliXXXX/proxy_arp = 1`）。這樣不管 Pod ARP 問誰，host 端都會用自己的 MAC 回應。Pod 拿到 MAC 後封好 L2 header，封包到了 host 端就進入 kernel 路由表，被當純 L3 封包來轉發。

所以 ARP 在這裡只是一個「騙 Pod 把封包送出來」的機制，實際轉發完全靠 L3 路由。

------

**第四層：跨 Node 怎麼辦？（BGP）**

Node A 上的 Pod 要跟 Node B 上的 Pod 通訊。Node A 的路由表需要知道「10.233.2.0/24 要走 Node B 的 IP」。這些路由怎麼來的？Calico 在每個 Node 上跑一個 **BIRD**（BGP daemon），Node 之間用 BGP 互相通告各自管的 Pod CIDR。這就是真正的「每個 Node 是一台 router」——它們之間跑 BGP 交換路由。

小規模時用 full-mesh BGP（每對 Node 互聯）。大規模時用 **Route Reflector**，避免 N² 的 BGP session。

------

**第五層：你問的「地址浪費」**

傳統做法是每個 Node 分一個固定的 /24（256 個 IP），但 Node 上可能只跑 10 個 Pod，剩下 246 個 IP 就浪費了。Calico 的 IPAM 設計可以緩解這個問題：它支援按需從 IP Pool 中分配更小的 block（預設 /26 = 64 個 IP），一個 Node 可以持有多個 block，也可以釋放空的 block 回 pool。所以浪費的粒度從 /24 縮小到 /26 甚至更細。但確實，block-based 分配天然會有一些碎片化的浪費，這是效率與路由表規模之間的 trade-off——block 越小浪費越少，但路由條目越多。

------

**第六層：安全（NetworkPolicy）**

Calico 用 iptables 或 eBPF 在每個 Node 上實現 NetworkPolicy。因為所有流量都經過 host 的 kernel 路由，Calico 可以在 veth 的入口和出口掛上規則，做細粒度的 Pod 間 ACL。這也是 Calico 很受歡迎的原因之一——它的 NetworkPolicy 實現比很多 CNI 更完整。

------

**總結框架（按順序消化）：**

1. **動機**：L3 純路由，不封裝 → 效能好、透明
2. **單機模型**：veth + 無 bridge + kernel 路由表 = Node 就是 router
3. **ARP 機制**：proxy ARP + link-local gateway → 騙 Pod 把封包送出來
4. **跨機模型**：BGP（BIRD）在 Node 間交換路由
5. **IPAM**：block-based 分配，/26 粒度，trade-off 在浪費 vs 路由表大小
6. **安全**：iptables/eBPF 實現 NetworkPolicy

你想先深入哪一層？我可以畫圖或用具體封包流程幫你走一遍。

## You said: 同樣的cillium來一份

同樣的cillium來一份

5月15日

## Claude responded: 好，同樣的框架結構，來拆解 Cilium。

好，同樣的框架結構，來拆解 Cilium。

Cilium 的核心思想一句話：**把網路的數據平面從 kernel 的傳統網路棧（iptables/netfilter）搬到 eBPF，直接在封包進入內核的最早期就做路由、過濾、負載均衡，跳過大量中間層。**

------

**第一層：為什麼要用 eBPF？（動機）**

傳統 CNI（包括 Calico 的 iptables 模式）依賴 netfilter/iptables 做封包處理。iptables 是一條條線性規則匹配，Pod 和 Service 數量一多，規則可以到上萬條，每個封包都要線性掃一遍，效能斷崖式下降。Cilium 的思路是：用 eBPF 把程式直接掛在 kernel 的網路 hook 點（tc ingress/egress、XDP、socket 層），用 hash map 做 O(1) 查表，不走 iptables 那套。結果是效能好、可程式化、而且可觀測性極強（因為 eBPF 天然能做 tracing）。

------

**第二層：Node 內部怎麼運作？（單機模型）**

跟 Calico 一樣，每個 Pod 有一個 veth pair。但封包的處理方式完全不同：

Cilium 在 host 端的 veth 上掛 eBPF 程式（tc BPF hook）。封包從 Pod 出來的瞬間就被 eBPF 程式攔截，這個程式直接查 BPF map（一個 kernel 內的 hash table）決定要怎麼處理——轉發、丟棄、NAT、還是送到 Service 的後端 Pod。整個過程**可以跳過 kernel 的 routing stack、跳過 netfilter、跳過 iptables**。如果目標 Pod 在同一個 Node 上，eBPF 甚至可以直接把封包從一個 veth redirect 到另一個 veth，不經過完整的網路棧。

------

**第三層：跟 Calico 對比——身份模型的根本差異**

Calico 的世界觀是 IP-centric：路由表、ACL 都是基於 IP 地址。

Cilium 的世界觀是 **Identity-centric**。每個 Pod（或一組 Pod）會被賦予一個 **Security Identity**（一個數字 ID），這個 ID 是由 Pod 的 label set 決定的。相同 label 的 Pod 共享同一個 Identity。eBPF 程式做 policy 判斷時查的不是「源 IP 是不是 10.233.1.5」，而是「源 Identity 是不是 12345」。好處是 IP 變了（Pod 重啟、漂移）policy 不受影響，而且不需要為每個 IP 生成一條規則。

------

**第四層：跨 Node 怎麼辦？（資料平面）**

Cilium 支援多種模式，按偏好排序：

第一種是 **Native Routing**，跟 Calico 類似，靠底層網路的路由把封包送到對的 Node。適合雲端（AWS ENI、GCP 路由）或自己有 BGP 的環境。沒有封裝開銷。

第二種是 **VXLAN/Geneve overlay**，當底層網路你控制不了（不能加路由）時，Cilium 會用 tunnel 封裝。但即使在 overlay 模式下，封裝和解封裝也是 eBPF 做的，不走 kernel 原生的 VXLAN 模組流程。

第三種是 **WireGuard 加密**，可以在任一模式上疊加透明加密，也是 eBPF 協助處理。

------

**第五層：Service 負載均衡（取代 kube-proxy）**

這是 Cilium 一個很大的賣點。傳統 K8s 用 kube-proxy 把 Service 的 ClusterIP 轉成後端 Pod IP，kube-proxy 靠 iptables 或 IPVS 實現。Cilium 可以完全取代 kube-proxy——用 eBPF 在 socket 層或 tc 層直接做 DNAT。一個封包到達時，eBPF 查 BPF map 裡的 Service → Endpoints 映射，O(1) 完成，不需要成千上萬條 iptables 規則。而且它可以在 **socket 層**（connect 系統呼叫時）就做轉換，封包從出生就帶著真實後端 IP，完全不需要中途 NAT 再反 NAT。

------

**第六層：可觀測性（Hubble）**

因為 eBPF 本身就是一個可程式的觀測點，Cilium 內建了 **Hubble**——一個網路流量的觀測平台。它可以看到每個封包的源 Identity、目標 Identity、是否被 policy 放行或丟棄、DNS 解析結果、HTTP method/path（L7 層級），全部都是 eBPF 在 kernel 裡直接抓取的，不需要 sidecar、不需要 tcpdump。

------

**第七層：NetworkPolicy（L3/L4 + L7）**

Cilium 支援標準 K8s NetworkPolicy，但更強的是它的 **CiliumNetworkPolicy** CRD，可以做到 L7 層級的控制。例如你可以寫一條規則說「label app=frontend 的 Pod 只能對 label app=api 的 Pod 發 GET /api/v1/users，不能 DELETE」。這在純 L3/L4 的方案（包括 Calico iptables 模式）裡做不到。實現方式是 eBPF 把封包送到 userspace 的 Envoy proxy 做 L7 解析，但只有命中 L7 policy 的流量才會走 proxy，其他的純 L3/L4 流量全在 kernel 裡 eBPF 處理完。

------

**跟 Calico 的對照表：**

| 維度        | Calico                    | Cilium                    |
| ----------- | ------------------------- | ------------------------- |
| 數據平面    | iptables 或 eBPF          | eBPF 為主                 |
| 路由方式    | BGP（BIRD）               | Native routing 或 overlay |
| Policy 基礎 | IP 地址                   | Identity（label-based）   |
| Service LB  | 依賴 kube-proxy           | 可取代 kube-proxy         |
| L7 Policy   | 需配合 Istio 等           | 內建（Envoy + eBPF）      |
| 可觀測性    | 需外部工具                | 內建 Hubble               |
| ARP 機制    | proxy ARP + link-local GW | eBPF 直接處理轉發         |
| 複雜度      | 相對簡單直觀              | 較複雜但功能更強          |

**總結框架（按順序消化）：**

1. **動機**：eBPF 取代 iptables → O(1) 查表、可程式化、可觀測
2. **單機模型**：veth + tc BPF hook → 封包一出 Pod 就被 eBPF 處理
3. **身份模型**：Identity（label hash）取代 IP-based → policy 不怕 IP 變動
4. **跨機模型**：native routing 或 VXLAN/Geneve overlay（都由 eBPF 處理）
5. **Service LB**：eBPF 在 socket/tc 層做 DNAT，取代 kube-proxy
6. **可觀測性**：Hubble，eBPF 原生抓取 L3-L7 流量數據
7. **安全**：CiliumNetworkPolicy，支援 L7 規則，eBPF + Envoy 混合處理

想深入哪一層？我可以畫 eBPF 的封包處理流程圖，或者拿一個具體封包走一遍 Cilium 和 Calico 的路徑對比。