Cilium 的設定不是「選一個模式」，而是五條獨立的設定軸，各自決定一件事，彼此可以自由組合。

**第一條軸：跨 Node routing**，決定封包離開本機之後怎麼到達另一台 Node。預設是 VXLAN 封裝模式，所有跨 Node 流量都包在 UDP 8472 裡面，好處是底層網路完全不需要知道 PodCIDR 的存在，部署最無腦，代價是每個封包多 ~50 bytes 的封裝開銷。Geneve 跟 VXLAN 概念一樣，只是協議換成 UDP 6081，支援可變長度的 metadata，Cilium 可以把 Identity 塞在裡面傳過去。第三個選項是 native routing，完全不封裝，封包直接用 Pod IP 路由出去，底層網路必須能路由 PodCIDR，通常搭配 BGP 或雲端的路由表來實現，效能最好但對基礎設施要求最高。

**第二條軸：kube-proxy replacement**，決定 Kubernetes Service 由誰處理。預設是保留 kube-proxy，Cilium 只管 CNI 和 Policy，Service 的 ClusterIP、NodePort 還是靠 iptables 或 IPVS。設成 partial 的話，Cilium 接管 ClusterIP 但 NodePort 還是留給 kube-proxy。設成 strict 就是完全移除 kube-proxy，所有 Service 流量由 BPF 處理。這時候你筆記裡寫的 `connect()` hook 才會生效 — Cilium 在 socket 層就把 ClusterIP 換成 backend Pod IP，所以 tcpdump 永遠抓不到 ClusterIP。

**第三條軸：同 Node host routing**，決定同一台機器上 Pod 之間封包走什麼路。預設是 legacy 模式，封包照常走 Linux host 的 network stack，經過 routing table 和 netfilter，相容性最好。另一個選項是 eBPF host routing，用 `bpf_redirect_peer()` 直接從 source Pod 的 veth 跳到 dest Pod 的 veth，完全不碰 host stack，延遲大幅降低，但需要 kernel 5.10 以上。你筆記裡畫的「封包完全不碰 Linux routing table」就是這個模式。

**第四條軸：IPAM**，決定 Pod IP 怎麼分配。預設是 Cluster Pool，Cilium 自己管理 IP pool 分配 PodCIDR 給每個 Node，最通用。Kubernetes 模式是讓 K8s 分配 PodCIDR，Cilium 從中取用，GKE 預設用這個。AWS ENI 模式讓 Pod 直接拿到 ENI 上的 IP，在 VPC 內原生可路由不需要 SNAT，但受限於 EC2 instance type 能掛多少 ENI。Azure 類似概念，Pod 拿 VNET 的 IP。

**第五條軸：傳輸加密**，決定 Node 之間的封包要不要加密。預設不加密，延遲最低。WireGuard 選項會在 Node 之間自動建立加密隧道，設定簡單效能也不錯，因為是 kernel 內建的。IPsec 是另一個選項，設定比較複雜，但某些合規場景會要求用 IPsec。

這五條軸是完全獨立的。你可以跑 VXLAN + strict kube-proxy replacement + eBPF host routing + Cluster Pool + WireGuard，也可以跑 native routing + 保留 kube-proxy + legacy host routing + AWS ENI + 不加密，任意排列組合都行。實務上常見的搭配大概是這幾種：最基本的預設安裝就是 VXLAN + 保留 kube-proxy + legacy + Cluster Pool + 不加密，什麼都不用改直接能跑。追求高效能的話會切成 native routing + strict + eBPF host routing，三條軸同時拉到最快。AWS 上跑 EKS 通常會用 native routing + strict + ENI + WireGuard。GKE 則是 native routing + strict + Kubernetes IPAM。


---



**① 跨 Node routing** — 封包怎麼送到另一台 Node
- VXLAN（預設）：UDP 8472 封裝，底層網路不用管 PodCIDR，最好部署，有封裝開銷
- Geneve：類似 VXLAN，UDP 6081，可攜帶更多 metadata
- Native routing：不封裝，靠底層網路路由 PodCIDR（BGP / 雲端路由表），效能最好

**② kube-proxy replacement** — Service 由誰處理
- Disabled（預設）：kube-proxy 照跑，Cilium 只管 CNI + Policy
- Partial：Cilium 接管 ClusterIP，NodePort 仍歸 kube-proxy
- Strict：完全移除 kube-proxy，BPF 在 socket 層就把 ClusterIP 換成 Pod IP

**③ 同 Node host routing** — 同機器 Pod 之間走什麼路
- Legacy（預設）：走 Linux host stack（routing table + netfilter）
- eBPF host routing：`bpf_redirect_peer()` 直接 veth 跳 veth，跳過 host stack，需 kernel 5.10+

**④ IPAM** — Pod IP 怎麼拿
- Cluster Pool（預設）：Cilium 自己管 IP pool
- Kubernetes：K8s 分配 PodCIDR，GKE 預設用這個
- AWS ENI：Pod 直接拿 ENI IP，VPC 內原生可路由
- Azure：Pod 拿 VNET IP

**⑤ 傳輸加密** — Node 之間要不要加密
- 無（預設）：明文傳輸，最低延遲
- WireGuard：kernel 內建，設定簡單效能好
- IPsec：合規場景使用，設定較複雜

**常見組合速查**
- 預設安裝：VXLAN + Disabled + Legacy + Cluster Pool + 無
- 高效能：Native + Strict + eBPF + Cluster Pool + 無
- AWS EKS：Native + Strict + eBPF + ENI + WireGuard
- GKE：Native + Strict + eBPF + K8s IPAM + 無

**Flannel** — 最簡單，幾乎沒什麼軸可以調：

- 跨 Node routing：只支援 VXLAN（有 DirectRouting 選項讓同子網走捷徑） [RKE2](https://docs.rke2.io/networking/basic_network_options)
- Dataplane：只有標準 Linux，沒有 eBPF
- kube-proxy replacement：不支援，kube-proxy 一定要跑
- IPAM：host-local 固定方式
- 加密：不內建
- Network Policy：不支援，需要另外裝 [RKE2](https://docs.rke2.io/networking/basic_network_options)

**Calico** — 跟 Cilium 最接近，幾乎每條軸都有對應選項：
- 跨 Node routing：VXLAN、IP-in-IP、Native BGP、CrossSubnet（同子網不封裝，跨子網才封裝）
- Dataplane：iptables（傳統）或 eBPF（v3.28+ 已接近 Cilium 效能）
- kube-proxy replacement：eBPF 模式下可以取代
- IPAM：Calico 自管、Kubernetes、AWS ENI
- 加密：WireGuard、IPsec

跟 Cilium 比，Calico 多了一個 IP-in-IP 封裝選項（比 VXLAN 開銷更小但不支援 eBPF 模式），而 BGP 原生路由是 Calico 的強項，歷史比 Cilium 更久更成熟

## 歸結學習路線
---
**同 Node Pod-to-Pod**
- 先懂 Linux 基礎：veth pair、network namespace、bridge（這是所有 CNI 的底層）
- 再懂 Cilium 怎麼跳過：TC hook 掛 BPF 程式、`bpf_redirect_peer()` 直接送到對面 veth
- 關鍵差異：傳統走 bridge + routing table + netfilter，Cilium 走 BPF 一步到位

**跨 Node Pod-to-Pod**
- Overlay 路線：VXLAN 封裝原理（外層 UDP 8472 + 內層原始封包）、Geneve 差異、MTU 開銷
- Underlay 路線：BGP 怎麼把 PodCIDR 廣播出去讓實體網路能路由、雲端路由表怎麼設
- 不用兩個都精通，看你環境選哪條就深入哪條

**Service 層**
- 先懂原生 kube-proxy：iptables 模式怎麼做 DNAT（ClusterIP → Pod IP）、IPVS 模式差在哪
- 再懂 Cilium 替換：BPF Service Map 用 hash O(1) 查表、cgroup `connect()` hook 在 socket 層就換掉目的地
- 關鍵差異：kube-proxy 是封包產生之後才改，Cilium 是封包產生之前就改

另外建議多加一層你沒提到的 — **Policy 層**，因為這是 Cilium 跟其他 CNI 拉開差距最大的地方：
- 傳統：NetworkPolicy 靠 iptables 實現，規則寫 IP
- Cilium：Identity-based，BPF Map 存的是 Identity 不是 IP，L7 還能透過 Envoy 做 HTTP path 層級的控制

所以完整的學習順序建議是：veth / namespace 基礎 → 同 Node datapath → 跨 Node（VXLAN 或 BGP 擇一）→ kube-proxy 原生 → Cilium Service 替換 → Identity + Policy。一層一層疊上去，每層都是在前一層的基礎上做優化。


所以完整的學習路徑可以這樣看：
```
① IPAM（Pod 怎麼拿到 IP）
② 同 Node Pod-to-Pod（veth、bridge、eBPF）
③ 跨 Node Pod-to-Pod（VXLAN / BGP）
④ Service（kube-proxy / Cilium 替換）
⑤ Ingress（外部怎麼進來）
⑥ Egress（Pod 怎麼出去）
⑦ Network Policy（誰能跟誰講話）
```

南北向要再分進來和出去：

**外部 → 進入 cluster（Ingress）**
- NodePort：每個 Node 開一個高位 port，外部直接打 Node IP:Port
- LoadBalancer：雲端幫你建一個 LB，分配外部 IP，轉發到 NodePort
- Ingress Controller / Gateway API：L7 層反向代理，根據 host / path 路由到不同 Service

**cluster → 出去外部（Egress）**
- SNAT / Masquerade：Pod IP 出去時換成 Node IP，外面看到的是 Node 的 IP
- Egress Gateway：指定特定 Node 作為出口，讓外部防火牆只需要放行固定 IP