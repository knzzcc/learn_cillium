# CKA / CKS 練習環境規劃指南

## 核心原則

> **底層相關 → Vagrant/VM**
> **應用層相關 → k3s**

判斷方式：問自己「這題的操作對象是 node/OS，還是 k8s 物件？」

---

## 環境分配對照表

| 主題 | 考試 | 建議環境 | 原因 |
|------|------|----------|------|
| CNI 安裝（Calico/Flannel） | CKA | Vagrant/VM | 需操作網路介面、iptables |
| kubeadm init / join / upgrade | CKA | Vagrant/VM | kubeadm 本身需要 bare node |
| etcd backup & restore | CKA | Vagrant/VM | 需存取 /var/lib/etcd、systemd |
| Node troubleshooting（kubelet 掛掉） | CKA | Vagrant/VM | systemctl restart kubelet |
| Static Pod 設定 | CKA | Vagrant/VM | /etc/kubernetes/manifests/ 路徑 |
| AppArmor / Seccomp | CKS | Vagrant/VM | 需要 OS 層 profile 路徑 |
| RBAC（Role、ClusterRole、SA） | CKA/CKS | k3s | 純 API 操作，k3s 完全支援 |
| NetworkPolicy | CKA/CKS | k3s | flannel + network policy plugin 可用 |
| PodSecurity / PSA | CKS | k3s | 純 API Server admission 設定 |
| Falco 規則偵測 | CKS | k3s | DaemonSet 部署即可練習 |
| Istio mTLS / AuthorizationPolicy | CKS | k3s | 需先停用 traefik |
| Secret 管理 / ImagePullSecret | CKA/CKS | k3s | 純物件操作 |
| Ingress / Service | CKA | 兩者皆可 | k3s 自帶 traefik ingress |
| PV / PVC / StorageClass | CKA | 兩者皆可 | k3s 有 local-path provisioner |

---

## 工具選擇

### Vagrant + VirtualBox vs VMware

**Vagrant + VirtualBox 優先**，原因如下：

- Vagrant 原生為 VirtualBox 設計，VirtualBox 是預設 provider
- 幾乎所有 Vagrantfile 範例、社群 box、troubleshooting 都針對 VB 環境
- VMware 需額外安裝 `vagrant-vmware-desktop` plugin，設定繁瑣
- VirtualBox 免費，VB + Vagrant 是最省事的組合

### Vagrant/VM 三台推薦配置

```
control-plane（1台）+ worker x2
```

- 直接用 kubeadm bootstrap
- 練完可 `vagrant destroy` 重來
- 每個主題 snapshot 一份，方便回滾

### k3s 適合「快速驗證概念」

- 啟動秒級，不用等 kubeadm
- 拿來練 RBAC / NetworkPolicy / PSA 效率高很多

---

## Istio 與 k3s

k3s 可以跑 Istio，但需注意：

```bash
# k3s 裝 Istio 前要先關掉 traefik（避免 port 衝突）
# /etc/rancher/k3s/config.yaml
disable:
  - traefik
```

Istio 在 k3s 上功能正常，適合練 mTLS、AuthorizationPolicy、PeerAuthentication（CKS 重點）。

---

## Vagrant Snapshot 策略

### 三台同步操作

snapshot 是針對單台 VM，若 cluster 跨三台，需同步操作，否則 etcd 狀態不一致會導致 cluster 壞掉。

```bash
# 三台同時 snapshot（建議先暫停 VM 再存）
vagrant snapshot save control-plane baseline
vagrant snapshot save worker1 baseline
vagrant snapshot save worker2 baseline

# restore 也要三台一起
vagrant snapshot restore control-plane baseline
vagrant snapshot restore worker1 baseline
vagrant snapshot restore worker2 baseline
```

### 建議存的時間點

| Snapshot 名稱 | 時機 |
|---------------|------|
| `baseline` | cluster 剛建好 + CNI 裝好（最重要） |
| `pre-etcd` | etcd backup/restore 練習前 |
| `pre-upgrade` | cluster upgrade 練習前 |

### Baseline 就夠用的題目（佔大多數）

- RBAC、NetworkPolicy、PodSecurity
- Pod / Deployment / Service 各種設定
- Secret、ConfigMap、PV/PVC
- Ingress
- Node taint / drain / cordon
- CertificateSigningRequest
- Falco、Audit log

### 需要額外 Snapshot 的情境（破壞性操作）

- etcd backup/restore（直接動 etcd 資料）
- cluster upgrade（版本升了就回不去）
- CNI 換掉或重裝
- control plane component 設定改壞

---

## 推薦 GitHub 資源

### 環境建置：techiescamp/vagrant-kubeadm-kubernetes

**用途：** 一鍵建立練習 cluster

```bash
git clone https://github.com/techiescamp/vagrant-kubeadm-kubernetes.git
cd vagrant-kubeadm-kubernetes
vagrant up
```

**注意事項：**
- 預設 k8s 版本為 1.31，需確認並對齊當前考試版本
- 修改 `settings.yaml` 調整版本：

```yaml
kubernetes_version: 1.35  # 依官方當前考試版本調整
```

### 練習題目：theplatformlab/CKA-Certified-Kubernetes-Administrator

**用途：** CKA 考試備考筆記與題目（2026 年 3 月考試，89 分）

**內容：**
- 31 個 hands-on exercises
- 2 份 mock exam（含完整解答）
- kubectl cheatsheet
- 23 份 YAML skeleton
- troubleshooting playbook
- 針對 v1.35 新功能說明（native sidecar、Gateway API）

**最佳搭配方式：**

```
techiescamp/vagrant-kubeadm-kubernetes  →  建立環境
theplatformlab/CKA-Certified-Kubernetes-Administrator  →  練習題目
```

---

## 推薦練習流程

1. 先在 k3s 理解概念（快速迭代）
2. 再到 Vagrant/VM 練「系統層」操作（etcd、CNI、kubeadm）
3. 用 [killer.sh](https://killer.sh) 模擬考試環境（購買 CKA/CKS 後附贈）
   - 建議一次在考前兩週，一次在考前三天

---

## 2026 考試重要資訊

| 項目 | 資訊 |
|------|------|
| Kubernetes 版本 | v1.35 |
| 考試時間 | 2 小時 |
| 及格分數 | CKA 66%、CKS 67% |
| 題目數量 | 約 17–25 題 |
| 考試費用 | $445 USD（含一次補考） |
| 考試方式 | PSI Secure Browser，終端機操作 |
| 可開啟文件 | kubernetes.io/docs、kubernetes.io/blog、github.com/kubernetes |

### v1.35 新增重點（舊資料沒有的）

| 功能 | 狀態 | 考試影響 |
|------|------|----------|
| Native sidecar containers | GA | init container 加 `restartPolicy: Always` |
| Gateway API | GA | 取代 Ingress，需會建 Gateway 和 HTTPRoute |
| kubectl debug | GA | `k debug node/<name>` 和 `k debug pod/<name>` |
| ValidatingAdmissionPolicy | GA | CEL-based admission，在課程表內 |
| cgroup v2 | Default | 影響資源監控與 limits |
