# 內容題目發想

## 核心定位

**K8s 攻防 × Cilium 防禦視角**

攻擊手法已有人寫（中國資安社群），防禦對比在台灣社群幾乎沒有。
固定角度：「這個攻擊手法，Cilium 能看到什麼、能防什麼」

---

## 參考來源

- [re0-kubernetes-sec-archive](https://github.com/neargle/re0-kubernetes-sec-archive) — 系統性 K8s 攻防知識庫，每一章都是潛在題目
- [freebuf K8s 攻擊點分析](https://www.freebuf.com/articles/network/342587.html) — 七大攻擊點，從攻擊者視角
- wu-andy HackMD — privileged=true 有多危險

---

## 題目清單

### K8s 網路系列（純網路，無安全角度）

- veth / network namespace 基礎
- 同 Node Pod-to-Pod（eBPF 怎麼繞過 kernel stack）
- 跨 Node（VXLAN vs native routing）
- Service（kube-proxy vs Cilium BPF Map）
- Cilium + Istio 封包路徑對比
- Ingress / Gateway API

### K8s 攻防 × Cilium 系列

| 攻擊手法 | Cilium 能加的視角 |
|---|---|
| privileged=true 容器逃逸 | Hubble 看到的異常流量長什麼樣 |
| ServiceAccount 濫用打 API server | L7 Policy 能擋嗎 |
| Cgroup release_agent 逃逸 | 網路層有沒有可觀測的痕跡 |
| etcd 未授權存取（port 2379） | NetworkPolicy 限制 etcd 只能被特定 Pod 存取 |
| kubelet 未授權（port 10250） | Cilium host firewall 怎麼鎖 |
| hostNetwork=true 裸奔 | tcpdump 能看到其他 Pod 的流量 |
| NodePort 暴露內部服務 | Egress Policy 怎麼限制 |

---

## 差異化說明

- 攻擊手法：中國資安社群已寫很多，台灣少
- 純 Cilium 說明：網路上有但不深
- **攻擊 + Cilium 防禦對比**：台灣社群目前沒有人在做

受眾是**正在學但搞不懂的人**，不是已經瞭如指掌的社群高手。
