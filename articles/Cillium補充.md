cillium怎麼知道我是跑在哪個模式，安裝的時候就指定了?

## Claude responded: 對，安裝的時候決定的。

對，安裝的時候決定的。

Cilium 預設用 **VXLAN（overlay 模式）**，不用特別指定就會用這個。如果你要換模式，安裝時帶參數：
```bash
# 預設 VXLAN overlay
cilium install

# 改用 Geneve overlay
cilium install --set tunnel=geneve

# 改用 native routing（直連模式，不封裝）
cilium install --set tunnel=disabled --set ipam.mode=kubernetes --set routingMode=native
```

幾個模式的差異：
- **VXLAN / Geneve（overlay）** — Pod 封包用隧道包起來跨節點，不用管底層網路怎麼設定，最省事。你的 VMware 環境用這個最適合。
- **Native routing（直連）** — 不封裝，效能好，但底層網路要能路由 Pod CIDR，設定比較麻煩。
- **eBPF kube-proxy replacement** — 用 eBPF 取代 kube-proxy，可以在任何模式上額外開啟。

你的環境用預設的 VXLAN 就好，直接 `cilium install` 不用加參數。等熟了再試其他模式，反正有 snapshot 可以還原。


我的cluster cillium就是想拿來玩ebpf阿 我現在vm cluster都接bridge可以用嗎? cilium install --set tunnel=disabled --set ipam.mode=kubernetes --set routingMode=native


可以用，Bridge 網路跑 Cilium native routing 沒問題。

但你這個參數組合是用 native routing（不走 tunnel），代表 Pod 的封包會直接走 Linux 的路由表，不包 VXLAN 或 Geneve。在 Bridge 網路下你需要讓每台 node 知道其他 node 的 Pod CIDR 怎麼走。

簡單來說兩條路：

**方案一：用 BGP 或手動加路由（真正的 native routing）**
每台 node 要加靜態路由指向其他 node 的 Pod 網段，比較麻煩但最接近真實環境，eBPF datapath 完整啟用。

**方案二：先用 tunnel 模式（建議先這樣）**
```bash
cilium install --set ipam.mode=kubernetes
```

預設就是 VXLAN tunnel，不用管路由，裝完就通。eBPF 的功能一樣全部都有，network policy、Hubble 觀察、kube-proxy replacement 這些都能玩。

tunnel 跟 native 的差別只在封包怎麼跨節點，eBPF 的核心功能兩種模式都一樣。你想玩 eBPF 不一定要 native routing。

建議先用 tunnel 模式把 cluster 跑起來，玩熟了再切 native routing 研究路由那塊。不然一開始就搞 native routing，踩坑又要多花時間。