# 學習目標（K8s 管理者 + 懂網路）

## 需要知道的

### 1. eBPF 是什麼（概念就好）
- hook point 在哪、BPF 程式怎麼被觸發
- BPF Map 是什麼（這是 Cilium 所有狀態的儲存核心）
- 不需要會寫 BPF 程式

### 2. Cilium 的核心概念
- **Endpoint**：Pod 在 Cilium 的管理單位
- **Identity**：從 Label 算出來的數字，取代 IP 作為 Policy 的基礎
- **ipcache**：IP → Identity 的查表，封包進來時靠這個知道「對方是誰」

### 3. Datapath 封包走哪條路
- 同 Node Pod-to-Pod：BPF 怎麼繞過 kernel stack 直接送達
- 跨 Node Pod-to-Pod：VXLAN overlay 的實際流程
- Pod-to-Service：DNAT 在 eBPF 層做掉，為什麼抓封包看不到 ClusterIP

### 4. Network Policy
- Identity-based Policy 怎麼運作
- L7 Policy（HTTP/gRPC）靠 Envoy 怎麼實現
- DNS-based Policy 怎麼做到

### 5. kube-proxy replacement
- eBPF 怎麼用 BPF Map hash lookup 取代 iptables O(n) 查表
- 各 Node 的 BPF Map 怎麼同步

### 6. 可觀測性（Hubble）
- 怎麼用 Hubble 看流量、確認 Policy 有沒有生效
- 出問題時怎麼用 `cilium monitor`、Hubble 定位問題

### 7. 進階功能（視需求）
- Service Mesh、Gateway API、Cluster Mesh
- 有用到再深入，沒用到看個大概即可

---

## 不需要知道的
- BPF C 原始碼細節
- Go 控制面程式碼
- Cilium 內部的 controller 機制
