# k3s vs k8s 差異

## 一句話總結

k3s 是「功能完整但換掉大型元件」的 k8s，
API 完全相容，kubectl 指令一模一樣，
差在底層元件的選擇和一些預設行為。

---

## 架構差異

### 標準 k8s

```
control plane:
  kube-apiserver       ← 獨立 process
  kube-scheduler       ← 獨立 process
  kube-controller-manager ← 獨立 process
  etcd                 ← 獨立 process（或外部 cluster）

worker:
  kubelet
  kube-proxy
  container runtime（containerd / CRI-O）
  CNI plugin（Calico / Flannel / Cilium...）
```

### k3s

```
control plane + worker:
  k3s server           ← 全部打包成一個 binary
  ├── apiserver
  ├── scheduler
  ├── controller-manager
  ├── SQLite（單節點）或 embedded etcd（HA）
  └── containerd（內建）

  flannel（內建 CNI）
  traefik（內建 Ingress）
  local-path-provisioner（內建 StorageClass）
```

k3s 把 control plane 所有元件合併成**一個 binary**，大幅減少安裝複雜度。

---

## 元件對比

| 項目 | k8s | k3s |
|------|-----|-----|
| 安裝方式 | kubeadm / 手動 | 單行 curl |
| 安裝時間 | 15-30 分鐘 | 1 分鐘 |
| 最低記憶體 | ~1.5GB | ~512MB |
| etcd | 獨立部署 | SQLite（單節點）/ embedded etcd（HA） |
| Container runtime | 自選（containerd/CRI-O） | 內建 containerd |
| CNI | 自選（Calico/Flannel/Cilium） | 內建 Flannel |
| Ingress | 自選安裝 | 內建 Traefik |
| StorageClass | 自選 | 內建 local-path-provisioner |
| LoadBalancer | 雲端或 MetalLB | 內建 ServiceLB |
| Binary 大小 | 多個 binary | 單一 ~100MB binary |

---

## 對 CKA/CKS 練習的影響

這是最重要的部分——哪些東西在 k3s 上**行為不同**或**不適合練**：

### ❌ k3s 不適合練的

**etcd 操作**
```bash
# k8s 的 etcd 是獨立 process
ETCDCTL_API=3 etcdctl snapshot save /tmp/backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt ...

# k3s 單節點用 SQLite，沒有 etcd process 可以操作
# HA 模式才有 embedded etcd，但路徑和憑證位置不同
```

**kubeadm**
```bash
# k3s 完全沒有 kubeadm，這些指令都不能用
kubeadm upgrade apply
kubeadm token create
kubeadm certs check-expiration
```

**Static Pod manifest 路徑**
```bash
# k8s
/etc/kubernetes/manifests/kube-apiserver.yaml

# k3s 沒有這些 static pod，apiserver 是 k3s binary 的一部分
# 修改 apiserver 設定要改 k3s service 的啟動參數
```

**CNI 從零安裝（Calico/Flannel）**
- k3s 內建 Flannel，CNI 已經裝好
- 想練裸裝 CNI 要用 VM/Vagrant

**kube-apiserver manifest 直接修改**
```bash
# k8s 改這個就能調 apiserver 參數
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml

# k3s 要改啟動參數，方式不同
sudo vi /etc/systemd/system/k3s.service
# 或用 /etc/rancher/k3s/config.yaml
```

---

### ✅ k3s 可以練的（行為完全一樣）

這些都是純 Kubernetes API 操作，k3s 完全支援：

```bash
# RBAC
kubectl create role / rolebinding / clusterrole / clusterrolebinding
kubectl auth can-i ...

# NetworkPolicy（需要 CNI 支援，Flannel 需額外裝 network policy）
kubectl apply -f networkpolicy.yaml

# Pod SecurityContext
securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  capabilities:
    drop: [ALL]

# Pod Security Admission
kubectl label ns <ns> pod-security.kubernetes.io/enforce=restricted

# Secrets / ConfigMap
kubectl create secret generic ...
kubectl create configmap ...

# PV / PVC / StorageClass
# k3s 內建 local-path provisioner，動態 PV 直接可用

# Deployment / Service / Ingress
# 完全一樣，Ingress 用內建 Traefik

# Audit logging
# 可以設定，但修改 apiserver 參數方式不同

# Falco
# 可以裝，行為完全一樣

# Trivy
# 掃 image，跟平台無關
```

---

## NetworkPolicy 在 k3s 的注意事項

k3s 預設的 Flannel **不支援 NetworkPolicy**，
需要額外裝 network policy controller：

```bash
# 方法 1：安裝時啟用（推薦）
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--flannel-backend=none \
  --disable-network-policy" sh -
# 然後自己裝 Calico 或 Cilium

# 方法 2：裝 k3s 後加裝 network policy
# 用 Cilium 取代 Flannel（支援 NetworkPolicy）
```

或直接用 Calico 取代 Flannel：

```bash
# 安裝時停用內建 CNI
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--flannel-backend=none \
  --disable-network-policy" sh -

# 裝 Calico
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

---

## Audit Logging 在 k3s 的設定方式

k8s 改 `/etc/kubernetes/manifests/kube-apiserver.yaml`，
k3s 改 `/etc/rancher/k3s/config.yaml`：

```yaml
# /etc/rancher/k3s/config.yaml
kube-apiserver-arg:
  - "audit-log-path=/var/log/k3s-audit.log"
  - "audit-policy-file=/etc/rancher/k3s/audit-policy.yaml"
  - "audit-log-maxage=30"
```

```bash
sudo systemctl restart k3s
```

---

## 實際使用建議

```
練習目標          用哪個
─────────────────────────────────────────
RBAC 概念驗證    → k3s（快，30 秒起）
NetworkPolicy    → k3s + Calico/Cilium
Pod Security     → k3s
Secrets 管理     → k3s
Ingress / TLS    → k3s（有內建 Traefik）
Falco / Trivy    → k3s
Helm / Kustomize → k3s

etcd backup      → Vagrant/VM
kubeadm upgrade  → Vagrant/VM
CNI 裸裝         → Vagrant/VM
kubelet 除錯     → Vagrant/VM
Static Pod 修改  → Vagrant/VM
```

---

## k3s 常見問題

**Q：k3s 的 kubeconfig 在哪？**
```bash
/etc/rancher/k3s/k3s.yaml
# 複製到 ~/.kube/config 或 export KUBECONFIG
```

**Q：怎麼看 k3s 版本對應的 k8s 版本？**
```bash
kubectl version
# k3s v1.35.0+k3s1 → kubernetes v1.35.0
```

**Q：k3s 重開機會自動啟動嗎？**
```bash
# 會，安裝時已設定 systemd service
systemctl is-enabled k3s   # 應該顯示 enabled
```

**Q：想完全移除 k3s？**
```bash
/usr/local/bin/k3s-uninstall.sh
```
