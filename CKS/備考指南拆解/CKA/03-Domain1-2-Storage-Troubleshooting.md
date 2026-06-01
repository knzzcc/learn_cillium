# Domain 1 — Storage（10%）

## PV / PVC 綁定條件

PVC 要綁定到 PV，三個條件必須全部符合：
1. capacity >= request
2. accessModes 相符
3. storageClassName **完全一致**（大小寫都算）

## Access Modes

| 縮寫 | 全名 | 說明 |
|------|------|------|
| RWO | ReadWriteOnce | 單一節點讀寫（最常用） |
| ROX | ReadOnlyMany | 多節點唯讀 |
| RWX | ReadWriteMany | 多節點讀寫（hostPath 不支援） |
| RWOP | ReadWriteOncePod | 單一 Pod 讀寫（v1.29+） |

## Reclaim Policy

| Policy | 行為 |
|--------|------|
| Retain | PVC 刪除後 PV 保留，需手動清理 |
| Delete | PVC 刪除後 PV 與底層儲存一起刪除 |
| Recycle | 已廢棄，不要用 |

## 基本 YAML

```yaml
# PV（cluster-scoped，無 namespace）
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /data/my-pv
---
# PVC（namespace-scoped）
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: manual
---
# Pod 掛載 PVC
volumes:
- name: data
  persistentVolumeClaim:
    claimName: my-pvc
```

## StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/no-provisioner
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

> `WaitForFirstConsumer`：等 Pod 被排程後才綁定，避免 PV 在 A 節點但 Pod 去 B 節點。

---

# Domain 2 — Troubleshooting（30%）

## Log 位置

| 元件 | 查法 |
|------|------|
| kubelet | `journalctl -u kubelet` |
| kube-apiserver | `k logs -n kube-system kube-apiserver-<node>` |
| etcd | `k logs -n kube-system etcd-<node>` |
| containerd | `journalctl -u containerd` |

## Pod 狀態對應處理

**Pending**
```bash
k describe pod <pod>    # 看 Events
k get pvc               # PVC 是否 Bound？
k describe node         # 有沒有 Taint？資源夠嗎？
```
常見原因：PVC 未綁定、node taint 無 toleration、資源不足、nodeSelector 沒有符合的節點

**CrashLoopBackOff**
```bash
k logs <pod> --previous   # 看上次掛掉的 log
```
常見原因：command/entrypoint 錯誤、缺少 env/secret/configmap

**ImagePullBackOff**
- 九成是 image 名稱拼錯
- 私有 registry 沒設 imagePullSecrets

**Pending（Service 無法連線）**
```bash
k get endpoints <svc>   # 是否為空？
k describe svc <svc>    # selector 是否對應到 pod labels？
```

## Node 無法連線

```bash
k get nodes
k describe node <node>
ssh <node>
sudo systemctl status kubelet
sudo journalctl -u kubelet --no-pager | tail -50
sudo systemctl restart kubelet

# 憑證過期？
sudo kubeadm certs check-expiration
sudo kubeadm certs renew all
```

## Control Plane 掛掉

```bash
# Static pod manifest 是否存在？
ls /etc/kubernetes/manifests/
# 應有：etcd.yaml、kube-apiserver.yaml、kube-controller-manager.yaml、kube-scheduler.yaml

# etcd 健康？
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

## 網路 Troubleshooting

```bash
# 1. Service selector 對不對？
k describe svc <svc>

# 2. Endpoints 有沒有 Pod？
k get endpoints <svc>

# 3. DNS 通嗎？
k run test --image=busybox:1.36 --rm -it -- nslookup <svc-name>

# 4. NetworkPolicy 擋了嗎？
k get networkpolicy -n <ns>

# 5. kube-proxy 有跑嗎？
k get pods -n kube-system -l k8s-app=kube-proxy
```
