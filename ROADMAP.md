# Cilium 學習路線圖

## 你的知識對應

| 你已知道 | Cilium 的解法 | 對應文章 |
|---|---|---|
| Pod-Pod 靠 veth | eBPF 在 veth 的 TC hook 直接 redirect，繞過 Linux network stack | Day12, Day13 |
| Node-Node 靠 VXLAN | Cilium 也支援 VXLAN，但多了 eBPF 加速，也可換成 native routing | Day14, Day15 |
| Service 靠 iptables，規則多效能差 O(n) | eBPF + BPF Map 做 hash lookup，O(1)，完全取代 kube-proxy | Day19, Day20 |

---

## 第一階段：理解基礎

**目標：搞清楚 eBPF 是什麼，Cilium 的關鍵概念**

- [ ] [Day01](articles/day01.md) — 痛點與動機（你已掌握，快速過）
- [ ] [Day04](articles/day04.md) — eBPF 概念篇：hook point 在哪、BPF 程式怎麼跑
- [ ] [Day05](articles/day05.md) — BPF Maps：eBPF 的記憶體資料結構，後面一切的核心
- [ ] [Day06](articles/day06.md) — 專有名詞：Endpoint、Identity、ipcache，不懂後面會迷路

---

## 第二階段：Pod 生命週期

**目標：一個 Pod 起來時，Cilium 背後做了什麼**

- [ ] [Day07](articles/day07.md) — CNI 呼叫流程，veth pair 的建立
- [ ] [Day08](articles/day08.md) — Cilium Agent 建立 Endpoint
- [ ] [Day09](articles/day09.md) — Identity 的誕生：基於 Label，不靠 IP，這是 Cilium 的核心創新
- [ ] [Day10](articles/day10.md) — BPF 程式與 Datapath 的成形，TC hook 被掛上去

---

## 第三階段：Datapath 深潛

**目標：一個封包怎麼從 A Pod 到 B Pod，每一步都看清楚**

- [ ] [Day11](articles/day11.md) — 工具箱：`cilium monitor`、`bpftool` 等除錯工具
- [ ] [Day12](articles/day12.md) — ARP 偽造：Cilium 為何要偽造 ARP？
- [ ] [Day13](articles/day13.md) — 同 Node Pod-to-Pod：eBPF 怎麼繞過 kernel stack 走捷徑
- [ ] [Day14](articles/day14.md) — `cilium_host`、`cilium_net`、`cilium_vxlan` 這些虛擬網卡是什麼
- [ ] [Day15](articles/day15.md) — 跨 Node 的封包路徑，VXLAN overlay 的實際流程

---

## 第四階段：Network Policy 與 L7

**目標：Cilium 的 Policy 比 K8s NetworkPolicy 強在哪**

- [ ] [Day16](articles/day16.md) — Identity-based Policy 入門
- [ ] [Day17](articles/day17.md) — Envoy 整合：L7 (HTTP/gRPC) Policy 怎麼實現
- [ ] [Day18](articles/day18.md) — DNS-based Policy：DNS Proxy 機制

---

## 第五階段：Service（kube-proxy replacement）

**目標：理解 kube-proxy replacement，iptables O(n) 問題的直接解答**

- [ ] [Day19](articles/day19.md) — 沒有 kube-proxy，eBPF 怎麼實現 Service LB
- [ ] [Day20](articles/day20.md) — 為什麼抓封包看不到 ClusterIP？（DNAT 在 eBPF 層就做了）
- [ ] [Day21](articles/day21.md) — 各 Node 的 BPF Map 如何同步

---

## 第六階段：進階功能

**目標：Service Mesh、Gateway API、跨叢集通信**

- [ ] [Day22](articles/day22.md) — Cilium Service Mesh 與 Envoy 流量分流
- [ ] [Day23](articles/day23.md) — Gateway API 入門與實作：取代 Ingress
- [ ] [Day24](articles/day24.md) — GAMMA Support：Gateway API 東西向流量
- [ ] [Day25](articles/day25.md) — Cluster Mesh 入門與配置
- [ ] [Day26](articles/day26.md) — Cluster Mesh 高可用與 NetworkPolicy 實踐

---

## 第七階段：實戰 Troubleshooting

**目標：真實踩坑案例，培養除錯直覺**

- [ ] [Day27](articles/day27.md) — 裝了 NodeLocalDNS Cache，DNS 封包原來都沒進去？
- [ ] [Day28](articles/day28.md) — 想監控 DNS，hubble_dns_queries_total 怎沒計算到？
- [ ] [Day29](articles/day29.md) — 裝了 Cilium 後，一個 Pod 要 Ready 需要等 26 分鐘？

---

## 建議起點

你的背景適合直接從這個順序開始：

1. **Day04** — 先懂 eBPF 的基本概念
2. **Day09** — 理解 Identity（Cilium 最核心的創新）
3. **Day13** — 看清楚封包路徑（目前已開啟的文章）
4. **Day19** — 了解 kube-proxy replacement，解答 iptables 效能瓶頸問題

---

## 閱讀策略（以 K8s 管理者 + 懂網路為目標）

### 主線：概念 + 整體運作

**第一步：eBPF 基礎概念**
```
Day01 → Day04 → Day05 → Day06
痛點    eBPF    BPF Maps  專有名詞
```

**第二步：Cilium 怎麼運作（遇到原始碼區塊直接略過）**
```
Day07 → Day08 → Day09 → Day10   （Pod 生命週期）
Day11 → Day12 → Day13 → Day14 → Day15   （Datapath）
Day16 → Day17 → Day18   （Network Policy）
Day19 → Day20 → Day21   （Service）
Day22 → Day23 → Day24 → Day25 → Day26   （進階功能）
```

### 實戰：除錯直覺

```
Day27 → Day28 → Day29   （真實踩坑案例）
```

### Source Code：不排進主線

遇到某個行為不理解時，再帶著問題去追對應文章的原始碼。管理者只需要知道「這個機制在哪個點介入」，不需要逐行讀懂。
