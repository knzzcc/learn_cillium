# k3s vs kind 比較

## 快速總結

| | **k3s** | **kind** |
|---|---|---|
| 全名 | Lightweight Kubernetes | Kubernetes IN Docker |
| 定位 | 輕量化的**真實** K8s 發行版 | 跑在 Docker 裡的**測試用** cluster |
| 底層 | 直接跑在 OS 上（binary） | Docker container 模擬節點 |
| 適合場景 | 邊緣設備、IoT、樹莓派、小型伺服器 | 本機開發、CI/CD、快速測試 |
| 多節點 | ✅ 真的多台機器 | ✅ 但都是假節點（同一台機器的 container） |
| 安裝速度 | 快（1 個指令） | 很快（需要先有 Docker） |
| 資源佔用 | 極低（512MB RAM 可跑） | 低，但比 k3s 稍高 |
| 像真實環境 | ✅ 非常接近（可用於生產環境） | 普通（純測試用） |
| Apple Silicon | ✅ 原生支援 | ✅ 原生支援 |
| Windows | ❌ 需要 VM 或 WSL2 | ✅ 原生支援 |
| CKA 練習適合度 | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| 持久化 Storage | ✅ 內建 local-path-provisioner | ❌ 需要額外設定 |
| LoadBalancer | ✅ 內建 ServiceLB（Klipper） | ❌ 需要 MetalLB |
| Ingress | ✅ 內建 Traefik | ❌ 需要自己安裝 |

---

## 安裝方式

### k3s

```bash
# 安裝 server（control plane）
curl -sfL https://get.k3s.io | sh -

# 加入 worker node
curl -sfL https://get.k3s.io | K3S_URL=https://<server-ip>:6443 K3S_TOKEN=<token> sh -

# 取得 kubeconfig
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

### kind

```bash
# 先安裝 kind
brew install kind  # macOS
# 或
go install sigs.k8s.io/kind@latest

# 建立單節點 cluster
kind create cluster

# 建立多節點 cluster（需要 config 檔）
kind create cluster --config kind-config.yaml
```

**kind 多節點設定範例（kind-config.yaml）：**

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

---

## 深入比較

### 1. 架構差異

**k3s** 把整個 Kubernetes 打包成單一 binary（< 100MB），移除了不常用的功能，並整合了 containerd、Flannel CNI、CoreDNS、Traefik Ingress、local-path-provisioner。

**kind** 把每個 K8s 節點跑在一個 Docker container 裡，節點之間透過 Docker network 溝通。本質上是「假」的多節點，所有節點其實都在同一台機器上。

### 2. 網路差異

| | k3s | kind |
|---|---|---|
| 預設 CNI | Flannel | kindnet（自研） |
| 自訂 CNI | ✅ 可停用預設後安裝 Calico/Cilium | ✅ 可在 config 停用後自行安裝 |
| NodePort 存取 | ✅ 直接存取 | ⚠️ 需要 `extraPortMappings` 設定 |
| LoadBalancer IP | ✅ 自動分配（ServiceLB） | ❌ 永遠 Pending，需要 MetalLB |

### 3. 儲存差異

| | k3s | kind |
|---|---|---|
| 預設 StorageClass | ✅ local-path（自動建立） | ❌ 無預設，需手動設定 |
| hostPath | ✅ | ✅ |
| 動態 Provisioning | ✅ 開箱即用 | ❌ 需額外安裝 |

### 4. 系統資源

| | k3s | kind |
|---|---|---|
| 最低 RAM | ~512 MB | ~1 GB（含 Docker overhead） |
| 磁碟空間 | ~200 MB binary | 依 Docker image 大小 |
| CPU | 極低 | 低 |

---

## 使用場景建議

### 選 k3s 的情況

- 跑在**樹莓派**或嵌入式裝置
- 需要**接近真實**的 K8s 環境
- **邊緣運算 / IoT** 部署
- 想要**持久化**的測試環境
- 學習 CKA，想練習 systemd、節點管理

### 選 kind 的情況

- **CI/CD pipeline**（GitHub Actions、GitLab CI）
- 需要**快速建立/刪除** cluster
- 只是想測試 YAML manifest
- **Windows** 開發環境
- 跑 **operator / controller** 的整合測試

---

## CKA 備考建議

> **最理想順序：kubeadm > k3s > kind**

| 環境 | 優點 | 缺點 |
|---|---|---|
| **kubeadm**（VirtualBox/Vagrant） | 最接近考試環境，練 etcd、CNI 安裝 | 需要較多硬體資源 |
| **k3s** | 輕量、接近真實操作 | 很多東西預設已整合，少練到手動設定 |
| **kind** | 最快建立，適合刷題 | 與考試環境差距較大 |

---

## 常用指令對照

| 操作 | k3s | kind |
|---|---|---|
| 建立 cluster | `curl -sfL https://get.k3s.io \| sh -` | `kind create cluster` |
| 刪除 cluster | `k3s-uninstall.sh` | `kind delete cluster` |
| 查看節點 | `kubectl get nodes` | `kubectl get nodes` |
| 取得 kubeconfig | `/etc/rancher/k3s/k3s.yaml` | `kind get kubeconfig` |
| 查看 cluster 狀態 | `systemctl status k3s` | `docker ps` |
| 載入本地 image | 需要 `k3s ctr images import` | `kind load docker-image <image>` |

---

## 參考資源

- [k3s 官方文件](https://docs.k3s.io/)
- [kind 官方文件](https://kind.sigs.k8s.io/)
- [Kubernetes 官方文件](https://kubernetes.io/docs/)
