# Domain 3 — Workloads & Scheduling（15%）

## Deployment 滾動更新

```bash
k set image deployment/webapp nginx=nginx:1.28
k rollout status deployment/webapp
k rollout undo deployment/webapp
k rollout undo deployment/webapp --to-revision=2
```

滾動更新策略：
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 多出幾個 Pod
      maxUnavailable: 0  # 同時最多幾個不可用
```

## ConfigMap / Secret 注入方式

```yaml
# 1. 全部 key 當 env
envFrom:
- configMapRef:
    name: app-config
- secretRef:
    name: db-creds

# 2. 單一 key 當 env
env:
- name: DB_USER
  valueFrom:
    secretKeyRef:
      name: db-creds
      key: DB_USER

# 3. 掛載為檔案
volumeMounts:
- name: config-vol
  mountPath: /etc/config
volumes:
- name: config-vol
  configMap:
    name: app-config
```

> 掛載為 volume 會取代整個目錄，只掛單一檔案用 `subPath`。

## Static Pod

```bash
# 找 staticPodPath
cat /var/lib/kubelet/config.yaml | grep staticPodPath
# 通常是 /etc/kubernetes/manifests/

# 建立 static pod
sudo vi /etc/kubernetes/manifests/my-pod.yaml

# 刪除 static pod（kubectl delete 無效，要刪檔案）
sudo rm /etc/kubernetes/manifests/my-pod.yaml
```

> Static pod 名稱會加上節點名，如 `my-pod-node1`。Control plane 元件都是 static pod。

## Pod 排程控制

```yaml
# nodeSelector（簡單）
spec:
  nodeSelector:
    disk: ssd

# Node affinity（彈性）
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disk
            operator: In
            values:
            - ssd

# Toleration（配合 taint 使用）
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

Taint effects：
- `NoSchedule`：新 Pod 不排進來
- `PreferNoSchedule`：盡量不排，但不強制
- `NoExecute`：現有 Pod 也驅逐

## Resource Requests / Limits

```yaml
resources:
  requests:
    cpu: "250m"     # scheduler 用來選節點
    memory: "64Mi"
  limits:
    cpu: "500m"     # 超過 → 限速
    memory: "128Mi" # 超過 → OOMKilled
```

---

# Domain 4 — Cluster Architecture, Installation & Configuration（25%）

## RBAC 速記

| 物件 | Scope | 搭配 |
|------|-------|------|
| Role | namespace | RoleBinding |
| ClusterRole | cluster | ClusterRoleBinding 或 RoleBinding |
| RoleBinding | namespace | Role 或 ClusterRole |
| ClusterRoleBinding | cluster | ClusterRole |

> ClusterRole + RoleBinding = 只在該 namespace 有效（不是 cluster-wide）

```bash
k create role pod-reader --verb=get,list,watch --resource=pods -n dev
k create rolebinding read-pods --role=pod-reader --serviceaccount=dev:my-sa -n dev
k auth can-i list pods -n dev --as=system:serviceaccount:dev:my-sa
```

## Kubeadm 建立 Cluster

```bash
# Control plane
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config

# 安裝 CNI
k apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Worker join（忘記指令時）
kubeadm token create --print-join-command
```

## Kubeadm Upgrade

```bash
# Control plane
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.35.0-1.1
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.35.0
sudo apt-get install -y kubelet=1.35.0-1.1 kubectl=1.35.0-1.1
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# Worker（先 drain）
k drain worker-1 --ignore-daemonsets --delete-emptydir-data
# ssh 進 worker
sudo apt-get install -y kubeadm=1.35.0-1.1 kubelet=1.35.0-1.1 kubectl=1.35.0-1.1
sudo kubeadm upgrade node
sudo systemctl daemon-reload && sudo systemctl restart kubelet
# 回 control plane
k uncordon worker-1
```

> control plane 用 `upgrade apply`，worker 用 `upgrade node`

## etcd Backup / Restore

```bash
# Backup
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Restore
ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored

# 修改 /etc/kubernetes/manifests/etcd.yaml
# --data-dir 與 volumes hostPath.path 改為 /var/lib/etcd-restored
```

> cert 路徑：`grep -E "cert|key|ca" /etc/kubernetes/manifests/etcd.yaml`
