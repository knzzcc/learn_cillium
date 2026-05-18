# K8s 容器除錯框架

> 核心邏輯：K8s 是一個多層系統，除錯的本質是「定位問題在哪一層，然後用那一層的工具去解決」。

---

## 框架總覽：五層模型

```
┌─────────────────────────────────────────────┐
│  L5  應用層     App 本身的邏輯、配置、依賴     │
├─────────────────────────────────────────────┤
│  L4  調度與編排層  Scheduler / Controller / etcd │
├─────────────────────────────────────────────┤
│  L3  節點管理層   Kubelet / CRI / Container Runtime │
├─────────────────────────────────────────────┤
│  L2  網路層      CNI / kube-proxy / DNS / Ingress  │
├─────────────────────────────────────────────┤
│  L1  OS 與 Kernel  cgroup / namespace / conntrack / fs │
└─────────────────────────────────────────────┘
```

除錯方向：**由上往下排除**。先確認不是 App 自己的問題，再逐層往下查。因為越上層的問題越常見、也越容易修。

---

## L5｜應用層

### 判斷訊號
- Pod 是 Running 但行為不正確（回應錯誤、處理慢）
- Liveness / Readiness probe 失敗
- CrashLoopBackOff（看 exit code）

### 排查路徑
1. `kubectl logs <pod>` 和 `kubectl logs <pod> --previous`（看崩潰前的日誌）
2. `kubectl describe pod`——看 Events 區段，注意 exit code：
   - **Exit 1**：App 自己拋錯
   - **Exit 137**：被 SIGKILL（通常是 OOMKilled）
   - **Exit 139**：SIGSEGV（segfault）
3. 確認 ConfigMap / Secret 有沒有掛對、環境變數對不對
4. 確認 resource requests/limits 是否合理——limits 太低就是在邀請 OOMKill

### 常見陷阱
- Readiness probe 路徑寫錯，Pod 永遠不進 Service endpoint，流量進不來但 Pod 本身是健康的
- App 啟動慢但 `initialDelaySeconds` 太短，變成起不來就被殺的循環

---

## L4｜調度與編排層

### 判斷訊號
- Pod 一直 Pending
- Deployment 更新卡住（rollout 不動）
- 資源明明夠但 Pod 就是不被調度

### 排查路徑
1. `kubectl describe pod`——看 Events 裡 Scheduler 的訊息：
   - `Insufficient cpu/memory`：request 超過可用資源
   - `node(s) had taint ... that the pod didn't tolerate`：taint/toleration 不匹配
   - `0/N nodes are available: N node(s) didn't match Pod's node affinity`：affinity 規則太嚴
2. `kubectl get nodes -o wide` 確認 node 狀態
3. etcd 健康狀態：`etcdctl endpoint health`、`etcdctl endpoint status`
   - etcd latency 高 → API server 回應慢 → 所有操作都慢
4. Controller manager 和 Scheduler 的 leader election：`kubectl get lease -n kube-system`

### 常見陷阱
- PDB（PodDisruptionBudget）設得太嚴，擋住了 drain 和 rollout
- ResourceQuota 在 namespace 層級限制了總量，即使 node 有資源也建不出 Pod
- etcd 跑在慢磁碟上，fsync latency 高，整個 control plane 變慢

---

## L3｜節點管理層

### 判斷訊號
- Node 變 NotReady
- Pod 卡在 ContainerCreating 或 Terminating
- `crictl ps` 與 `kubectl get pods` 狀態不一致

### 排查路徑
1. `systemctl status kubelet` + `journalctl -u kubelet -e`
   - **PLEG is not healthy**：kubelet 來不及處理 container 事件，常見於 node 上 Pod 太多或 runtime 回應慢
   - **failed to garbage collect**：image 或 container GC 卡住
2. `crictl ps` / `crictl pods`——確認 runtime 層看到的狀態
3. `crictl info`——看 containerd 本身的健康狀態
4. Container 建不出來時，`crictl logs <container-id>` 或看 `/var/log/containers/`

### 穿透抽象層找 PID（前面討論的重點）
- `crictl inspect <container-id>` → 取得 PID
- 備選：`/sys/fs/cgroup/.../cgroup.procs`、遍歷 `/proc/<pid>/cgroup`
- 拿到 PID 後：`nsenter -t <pid> -n -p -m` 進入對應 namespace 操作

### 權限卡關（前面討論的重點）
- seccomp profile 擋 syscall → 報 EPERM
- capabilities 不足 → 加對應的 capability 或從 host 側 nsenter
- AppArmor / SELinux → 查 `dmesg` 裡的 DENIED 訊息
- PSA label 擋住 debug pod → 用 `kubectl debug node/` 繞過

---

## L2｜網路層

### 判斷訊號
- Pod IP 互通但 Service ClusterIP 不通 → kube-proxy 問題
- 跨 Node 的 Pod 互不通 → CNI / overlay 問題
- DNS 解析慢或失敗 → CoreDNS 問題
- 外部流量進不來 → Ingress 問題

### 排查路徑

**kube-proxy / Service 層**
1. `iptables-save | grep <clusterIP>` 或 `ipvsadm -Ln`——規則在不在
2. `kubectl get endpoints <service>`——endpoint 列表有沒有對應的 Pod IP
3. endpoint 為空 → Pod 的 label 跟 Service selector 沒對上，或 Readiness probe 沒過

**CNI 層**
1. `ip link` / `ip route` 看 node 上的網路介面和路由表
2. 跨 node ping Pod IP——不通就是 CNI 的問題
3. 看 CNI 的 Pod 日誌（Calico: `calico-node`、Flannel: `kube-flannel`、Cilium: `cilium-agent`）
4. Calico 常見：BGP peer 狀態 `calicoctl node status`
5. IP 池耗盡：`kubectl get ipamblocks`（Calico）或看 CNI 日誌的 IP exhaustion 訊息

**DNS 層**
1. `kubectl exec <pod> -- nslookup kubernetes.default`——最基本的 DNS 測試
2. CoreDNS Pod 狀態：`kubectl get pods -n kube-system -l k8s-app=kube-dns`
3. Node 上的 `/etc/resolv.conf` 有沒有被改
4. `ndots` 設定（預設 5）會讓每次查詢產生多次無效查詢，高 QPS 時可能拖垮 CoreDNS

**Ingress 層**
1. `kubectl describe ingress`——看 rules 和 backend 是否正確
2. Ingress controller 的 Pod 日誌
3. 確認 TLS secret 存在且沒過期

---

## L1｜OS 與 Kernel 層

### 判斷訊號
- 以上各層都正常但問題依舊存在
- `dmesg` 出現 kernel 層級的錯誤訊息
- 問題跟 node 的運行時間正相關（跑越久越嚴重）

### 排查路徑與常見問題

**cgroup 相關**
- **kmem leak**（kernel 3.x-4.x）：cgroup 刪除後 kmem 資源不釋放，cgroup 變殭屍。`cat /proc/cgroups` 裡 memory 的 num_cgroups 持續增長不回收。解法：升級 kernel 或關閉 kmem accounting
- **PID 耗盡**：`kernel.pid_max` 或 cgroup pids.max 到頂，`fork()` 回傳 EAGAIN。`cat /proc/sys/kernel/pid_max` 確認上限

**網路 kernel 資源**
- **conntrack 表滿**：`dmesg` 出現 `nf_conntrack: table full, dropping packet`。`sysctl net.netfilter.nf_conntrack_max` 查上限，`conntrack -C` 看目前使用量
- **socket buffer / backlog**：`netstat -s` 看 drop 統計

**檔案系統 / inode**
- **inotify watch 耗盡**：`fs.inotify.max_user_watches` 預設值在大量 Pod 的 node 上不夠用
- **inode 耗盡**：`df -i` 顯示 inode 100%，磁碟空間還有但建不了新檔案。常見於大量小檔案的 log 目錄

**記憶體**
- **OOM Killer**：`dmesg | grep -i oom`，看被殺的是 container process 還是系統關鍵 process（如 kubelet）
- **memory cgroup 的 swap 行為**：kernel 和 cgroup 的 swap 設定不一致會造成 OOM 計算不準

---

## 除錯決策流程

```
問題出現
  │
  ├─ Pod 層面有異常嗎？（CrashLoop / Error / Pending / Terminating）
  │   ├─ CrashLoop / Error → L5（看 logs 和 exit code）
  │   ├─ Pending → L4（看 describe 的 scheduling 訊息）
  │   ├─ ContainerCreating 卡住 → L3（看 kubelet 日誌和 runtime 狀態）
  │   └─ Terminating 卡住 → L3（kubelet 能不能正常操作）或 L1（cgroup 清理問題）
  │
  ├─ Pod Running 但行為異常？
  │   ├─ 連線問題 → L2（按 Service / CNI / DNS / Ingress 逐層排查）
  │   ├─ 效能問題 → L5 先確認 App，再 L1 看 kernel 資源
  │   └─ 間歇性問題 → L1 優先（conntrack / PID / inotify 等資源耗盡通常是間歇性的）
  │
  └─ Node 層面異常？（NotReady / 回應慢）
      ├─ kubelet 狀態 → L3
      ├─ etcd 慢 → L4
      └─ dmesg 有 kernel 錯誤 → L1
```

---

## 工具速查

| 層級 | 工具 | 用途 |
|------|------|------|
| L5 | `kubectl logs`, `kubectl describe`, `kubectl exec` | 看 App 日誌、事件、進入容器 |
| L4 | `etcdctl`, `kubectl get events --sort-by=.lastTimestamp` | etcd 健康、cluster 事件 |
| L3 | `crictl`, `journalctl -u kubelet`, `nsenter` | runtime 狀態、kubelet 日誌、進入 namespace |
| L2 | `iptables-save`, `ipvsadm`, `calicoctl`, `nslookup`, `tcpdump` | 網路規則、CNI 狀態、DNS、抓包 |
| L1 | `dmesg`, `sysctl`, `/proc/`, `/sys/fs/cgroup/`, `perf`, `strace` | kernel 訊息、系統參數、底層追蹤 |
