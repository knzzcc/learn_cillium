# K8s Cluster 建置指南

> **環境：** Windows + Ubuntu Desktop (VMware Host) + 3 台 Ubuntu Server VM  
> **網段：** 192.168.137.0/24

---

## 架構

```
Windows (192.168.137.1)
    │ 網路線
Linux Host - Ubuntu Desktop (192.168.137.150)
    │ VMware Bridge
    ├── master  (192.168.137.101) - Control Plane
    ├── worker1 (192.168.137.102) - Worker Node
    └── worker2 (192.168.137.103) - Worker Node
```

---

## 1. VM 規格

| 節點    | CPU | RAM  | 磁碟 | OS                  |
| ------- | --- | ---- | ---- | ------------------- |
| master  | 2   | 4 GB | 30GB | Ubuntu Server 22.04 |
| worker1 | 2   | 4 GB | 30GB | Ubuntu Server 22.04 |
| worker2 | 2   | 4 GB | 30GB | Ubuntu Server 22.04 |

---

## 2. Template VM 基礎設定（三台共用，clone 前做好）

### 2.1 更新系統

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl apt-transport-https ca-certificates openssh-server net-tools
```

### 2.2 設定時區

```bash
sudo timedatectl set-timezone Asia/Taipei
```

### 2.3 禁用 cloud-init 網路管理

```bash
echo "network: {config: disabled}" | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

### 2.4 關閉 Swap

```bash
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

### 2.5 載入 Kernel Modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### 2.6 設定 Sysctl 參數

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### 2.7 安裝 containerd

```bash
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 2.8 安裝 kubeadm、kubelet、kubectl

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 2.9 關機作為 Template

```bash
sudo shutdown -h now
```

---

## 3. Clone 並設定各節點

### 3.1 Clone VM（在 Linux Host 執行）

```bash
vmrun clone ~/vmware/template/template.vmx ~/vmware/master/master.vmx full -cloneName=master
vmrun clone ~/vmware/template/template.vmx ~/vmware/worker1/worker1.vmx full -cloneName=worker1
vmrun clone ~/vmware/template/template.vmx ~/vmware/worker2/worker2.vmx full -cloneName=worker2
```

### 3.2 修改 VM 規格（編輯各 .vmx）

```bash
#!/bin/bash
declare -A VMS
VMS[master]="2 4096"
VMS[worker1]="2 4096"
VMS[worker2]="2 4096"

for name in "${!VMS[@]}"; do
  read cpu ram <<< "${VMS[$name]}"
  VMX=~/vmware/$name/$name.vmx
  sed -i "s/^numvcpus.*/numvcpus = \"$cpu\"/" "$VMX"
  sed -i "s/^memsize.*/memsize = \"$ram\"/" "$VMX"
  sed -i "s/^displayName.*/displayName = \"$name\"/" "$VMX"
  sed -i "s/^ethernet0.connectionType.*/ethernet0.connectionType = \"bridged\"/" "$VMX"
  vmrun start "$VMX" nogui
done
```

### 3.3 設定各節點 Hostname

```bash
# master
sudo hostnamectl set-hostname master

# worker1
sudo hostnamectl set-hostname worker1

# worker2
sudo hostnamectl set-hostname worker2
```

### 3.4 設定各節點網路

每台編輯 `/etc/netplan/01-static.yaml`：

**master：**

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.137.101/24
      routes:
        - to: default
          via: 192.168.137.1
      nameservers:
        addresses:
          - 8.8.8.8
```

**worker1** → 改 `192.168.137.102/24`  
**worker2** → 改 `192.168.137.103/24`

```bash
sudo netplan apply
```

### 3.5 設定 /etc/hosts（三台都加）

```bash
cat <<EOF | sudo tee -a /etc/hosts
192.168.137.101 master
192.168.137.102 worker1
192.168.137.103 worker2
EOF
```

---

## 4. 設定 SSH 免密碼登入

### 從 Windows：

```cmd
ssh-keygen -t ed25519
ssh-copy-id kano@192.168.137.101
ssh-copy-id kano@192.168.137.102
ssh-copy-id kano@192.168.137.103
```

### 從 Linux Host：

```bash
ssh-keygen -t ed25519
ssh-copy-id kano@192.168.137.101
ssh-copy-id kano@192.168.137.102
ssh-copy-id kano@192.168.137.103
```

---

## 5. 初始化 K8s Cluster（在 master 執行）

### 5.1 kubeadm init

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.137.101
```

### 5.2 設定 kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 5.3 記下 join 指令

init 完成後會輸出類似以下指令，先複製起來：

```
kubeadm join 192.168.137.101:6443 --token xxxxx --discovery-token-ca-cert-hash sha256:xxxxx
```

---

## 6. 加入 Worker 節點（在 worker1、worker2 執行）

```bash
sudo kubeadm join 192.168.137.101:6443 \
  --token xxxxx \
  --discovery-token-ca-cert-hash sha256:xxxxx
```

如果 token 過期，在 master 重新產生：

```bash
kubeadm token create --print-join-command
```

---

## 7. 安裝 CNI（在 master 執行）

### 方案 A：Cilium

```bash
# 安裝 Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz

# 安裝 Cilium
cilium install

# 等待就緒
cilium status --wait

# 驗證連線
cilium connectivity test
```

### 方案 B：Calico

```bash
# 安裝 Tigera Operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/tigera-operator.yaml

# 套用 Calico 設定
cat <<EOF | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF
```

---

## 8. 驗證 Cluster

```bash
# 所有節點應為 Ready
kubectl get nodes

# 所有 system pod 應為 Running
kubectl get pods -A

# 測試部署
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get svc nginx
```

用瀏覽器打開 `http://192.168.137.101:<NodePort>` 確認看到 nginx 歡迎頁面。

---

## 9. 拍 Snapshot（在 Linux Host 執行）

```bash
vmrun snapshot ~/vmware/master/master.vmx "k8s-ready"
vmrun snapshot ~/vmware/worker1/worker1.vmx "k8s-ready"
vmrun snapshot ~/vmware/worker2/worker2.vmx "k8s-ready"
```

弄壞了隨時還原：

```bash
vmrun revertToSnapshot ~/vmware/master/master.vmx "k8s-ready"
```

---

## 10. 從 Windows 操作 Cluster（可選）

### 10.1 安裝 kubectl

```powershell
winget install Kubernetes.kubectl
```

### 10.2 複製 kubeconfig

從 master 拿 config：

```bash
cat ~/.kube/config
```

貼到 Windows 的 `%USERPROFILE%\.kube\config`，把 `server:` 的 IP 改成 `192.168.137.101`。

### 10.3 驗證

```powershell
kubectl get nodes
```

---

## 11. VM 開機自啟（在 Linux Host 設定）

```bash
sudo nano /etc/systemd/system/vmware-vms.service
```

```ini
[Unit]
Description=Start VMware VMs
After=network.target vmware.service

[Service]
Type=oneshot
RemainAfterExit=yes
User=kano
Group=kano
ExecStart=/usr/bin/vmrun start /home/kano/vmware/master/master.vmx nogui
ExecStart=/usr/bin/vmrun start /home/kano/vmware/worker1/worker1.vmx nogui
ExecStart=/usr/bin/vmrun start /home/kano/vmware/worker2/worker2.vmx nogui
ExecStop=/usr/bin/vmrun stop /home/kano/vmware/master/master.vmx
ExecStop=/usr/bin/vmrun stop /home/kano/vmware/worker1/worker1.vmx
ExecStop=/usr/bin/vmrun stop /home/kano/vmware/worker2/worker2.vmx

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable vmware-vms.service
```

---

## 常見問題

| 問題 | 解決方式 |
| --- | --- |
| netplan apply 重開機後失效 | 禁用 cloud-init 網路管理 |
| VMware kernel module 編譯失敗 | 降 kernel 到 6.8 或用社群 patch |
| vmrun 報 file already in use | 刪除 .lck 資料夾 |
| node 狀態 NotReady | 確認 CNI 已安裝且 pod 都 Running |
| token 過期無法 join | master 執行 `kubeadm token create --print-join-command` |
