# Cilium 核心概念筆記

## 2. Cilium 核心概念

### Endpoint
Pod 在 Cilium 眼中的抽象單位。每個 Pod 起來，Cilium 就建一筆 Endpoint 記錄，裡面存著這個 Pod 的 IP、ID、Identity、對應的 BPF 程式掛在哪個 veth。

```
kubectl get ciliumendpoint
NAME     SECURITY IDENTITY   ENDPOINT STATE   IPV4
client   26328               ready            10.244.2.16
server   9764                ready            10.244.2.135
```

### Identity
Cilium 用來識別「這個封包是誰發的」的數字，從 Pod 的 Labels 計算出來。

```
Pod A  labels: app=frontend  →  Identity: 26328
Pod B  labels: app=frontend  →  Identity: 26328  ← 同一組，同一個數字
Pod C  labels: app=backend   →  Identity: 9764
```

**為什麼要這樣設計？**
傳統 iptables 靠 IP 寫規則，Pod 換掉 IP 就變了，規則要重寫。
Cilium 靠 Identity，只要 Label 沒變，Identity 就不變，BPF Map 裡的規則一行都不用動。

### ipcache
一張「IP → Identity」的對照表，存在 BPF Map 裡。封包進來時，Cilium 查這張表知道對方的 Identity 是什麼，再去比對 Policy 決定要不要放行。

```
10.244.2.16  →  Identity: 26328
10.244.1.9   →  Identity: 9764
```

---

## 3. Datapath 封包走哪條路

### 同 Node Pod-to-Pod

傳統方式封包要過一遍 Linux kernel network stack，Cilium 直接跳過：

```
client Pod
  ↓ veth（TC ingress hook）
  BPF 程式攔截 → 查 ipcache 確認目的地是同 Node
  ↓ 直接 redirect 到 server Pod 的 veth（不經過 kernel routing）
server Pod
  ↓ veth（TC ingress hook）
  BPF 程式做 ingress policy 檢查
  ↓ 放行 → server Pod 收到
```

關鍵：封包完全不碰 Linux routing table，直接從一個 veth 跳到另一個 veth。

### 跨 Node Pod-to-Pod（VXLAN）

```
client Pod（worker-1）
  ↓ veth TC hook
  BPF 判斷目的地在不同 Node → 要走 overlay
  ↓ cilium_vxlan 封裝（外層加 UDP 8472，內層是原始封包）
  ↓ 走實體網路到 worker-2
worker-2 的 cilium_vxlan
  ↓ TC hook 解封裝
  ↓ 送到 server Pod 的 veth
server Pod 收到
```

### Pod-to-Service

Cilium 在 connect() 系統呼叫那一刻就把 ClusterIP 換掉了：

```
應用程式呼叫 connect(ClusterIP:80)
  ↓ cgroup hook 攔截（還沒建立 socket）
  BPF 查 Service Map → 選一個 backend Pod
  直接把 socket 的目的地改成 backend Pod IP
  ↓ TCP SYN 封包出來時，dst 已經是 Pod IP
```

所以用 tcpdump 抓封包，永遠看不到 ClusterIP，因為它在封包產生之前就被換掉了。

---

## 4. Network Policy

### Identity-based Policy 怎麼運作

規則寫的是「哪個 Identity 可以連哪個 Identity」，存在 BPF Map 裡。
封包進來時，BPF 程式查 ipcache 拿到來源 Identity，再查 Policy Map 決定放行或 drop。

```yaml
# 只允許 app=client 連 app=server 的 80 port
endpointSelector:
  matchLabels:
    app: server
ingress:
  - fromEndpoints:
    - matchLabels:
        app: client
    toPorts:
      - ports:
        - port: "80"
```

### L3/L4 vs L7 行為差異

| 情況 | 執行層 | 結果 |
|---|---|---|
| Identity 不符合 | BPF 直接 drop | 連線 timeout（無聲丟棄） |
| Identity 符合但 HTTP path 不對 | Envoy L7 proxy | 立即回 `Access denied` |

### L7 Policy 怎麼實現
Cilium 在每個 Node 上有一個 Envoy 進程。當 Policy 有 L7 規則時，BPF 程式把封包 redirect 到本機的 Envoy，由 Envoy 做 HTTP/gRPC 層的規則判斷，再決定放行或拒絕。

### DNS-based Policy
你可以寫 `toFQDNs: github.com` 這樣的規則。Cilium 有一個 DNS Proxy，Pod 的 DNS 查詢會先過 DNS Proxy，Proxy 記下 domain 對應的 IP，再動態更新 Policy 讓這些 IP 被放行。

---

## 5. kube-proxy replacement

### 問題根源
kube-proxy iptables 模式：每個封包要從頭到尾比對所有規則，Service 越多規則越多，時間複雜度 O(n)。

### Cilium 的解法
用 BPF Map（hash table）取代 iptables。查表時間複雜度 O(1)，Service 數量再多也不影響效能。

```
iptables：規則 1 → 規則 2 → 規則 3 → ... → 規則 n（逐條比對）
BPF Map：(ClusterIP, Port) 直接 hash → 找到 backend（一步到位）
```

### 各 Node 的 BPF Map 怎麼同步
每個 Node 的 Cilium Agent 都有一份自己的 BPF Map。
同步機制靠兩個東西：
- **KVStore（etcd）**：存 Identity、跨 Node 的 Policy 資訊
- **K8s API Watch**：Watch Service / Endpoint 變動，各 Node 的 Agent 收到通知後各自更新自己的 BPF Map
