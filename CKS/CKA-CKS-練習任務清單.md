# CKA / CKS 練習任務清單

> 標記說明：
> - 🔴 不熟 — 要認真練
> - 🟡 有印象 — 快速複習補細節
> - 🟢 熟 — 跳過或只驗證

---

## 一、k8s 底層（Vagrant/VM）

### 🔴 etcd Backup & Restore
**任務：**
1. 在 cluster 建立幾個 deployment 作為基準
2. 執行 etcd snapshot backup 到 `/tmp/etcd-backup.db`
3. 驗證 snapshot 狀態
4. 刪掉所有 deployment
5. 從 snapshot restore，確認 deployment 回來了

**驗證：**
```bash
ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup.db --write-table
kubectl get deployments   # restore 後要回來
```

---

### 🔴 kubelet 除錯
**任務：**
1. SSH 進 worker node
2. 手動停掉 kubelet：`sudo systemctl stop kubelet`
3. 從 control plane 觀察 node 狀態變化
4. 重新啟動 kubelet，觀察恢復
5. 故意改壞 kubelet config（加一個不存在的 flag），看 journalctl 的錯誤訊息，再修回來

**驗證：**
```bash
kubectl get nodes           # 應該看到 NotReady
sudo journalctl -u kubelet -f   # 看錯誤
```

---

### 🔴 kubeadm Cluster Upgrade
**任務：**
1. 確認目前版本
2. 升級 control plane（kubeadm → kubelet → kubectl）
3. drain worker node
4. 升級 worker
5. uncordon worker

**驗證：**
```bash
kubectl get nodes   # 版本應更新
kubectl get pods -A # 所有 pod 應該正常
```

---

### 🔴 Static Pod 建立與修改
**任務：**
1. 找到 staticPodPath 路徑
2. 手動建立一個 static pod manifest
3. 確認 pod 自動出現（名稱會加上節點名）
4. 故意改壞 kube-apiserver.yaml 的一個 flag，觀察 apiserver 起不來
5. 修回來，確認恢復

**驗證：**
```bash
cat /var/lib/kubelet/config.yaml | grep staticPodPath
kubectl get pod -A | grep <node-name>
```

---

### 🔴 Certificate 管理
**任務：**
1. 檢查所有憑證到期時間
2. 更新所有憑證
3. 確認 kubeconfig 還能用

**驗證：**
```bash
sudo kubeadm certs check-expiration
kubectl get nodes   # 更新後還能存取
```

---

### 🟡 Node 管理（drain / cordon / taint）
**任務：**
1. cordon 一個 worker，確認新 pod 不會排進去
2. drain worker（搬走 pod），確認 pod 移到其他節點
3. uncordon 恢復
4. 對 node 加 taint，建一個有對應 toleration 的 pod，確認排進去
5. 建一個沒有 toleration 的 pod，確認被擋

**驗證：**
```bash
kubectl get nodes             # SchedulingDisabled
kubectl describe node <node>  # 看 taint
kubectl get pods -o wide      # 確認排程位置
```

---

## 二、API Server 加固

### 🔴 Audit Logging
**任務：**
1. 寫一份 audit policy（secret 記 Metadata、pod 記 RequestResponse、event 不記）
2. 修改 kube-apiserver manifest 加入 audit flags 和 volume mount
3. 等 apiserver 重啟
4. 建立一個 secret，在 audit log 確認有記錄
5. 刪掉 secret，確認也有記錄

**驗證：**
```bash
sudo cat /var/log/kubernetes/audit/audit.log | jq 'select(.objectRef.resource=="secrets")'
```

---

### 🔴 API Server 加固 flags
**任務：**
1. 確認以下 flags 都有設定：
   - `--anonymous-auth=false`
   - `--authorization-mode=Node,RBAC`
   - `--profiling=false`
   - `--insecure-port=0`
2. 測試 anonymous 存取被擋
3. 用 kube-bench 掃描確認分數

**驗證：**
```bash
curl -k https://localhost:6443/api/v1/pods   # 應該 401
kube-bench run --targets=master
```

---

### 🔴 Secrets 加密靜態資料（EncryptionConfiguration）
**任務：**
1. 產生 32 bytes 的 base64 key
2. 建立 EncryptionConfiguration YAML
3. 修改 kube-apiserver manifest 加入 `--encryption-provider-config`
4. 建立一個 secret，直接查 etcd 確認是加密的
5. 重新加密所有現有 secret

**驗證：**
```bash
# 直接查 etcd 看 secret value 是否已加密（不是明文）
ETCDCTL_API=3 etcdctl get /registry/secrets/default/my-secret \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=... --cert=... --key=... | hexdump -C | head
# 應該看到 k8s:enc:aescbc 開頭
```

---

## 三、RBAC

### 🔴 最小權限設計
**任務：**
1. 建立 ServiceAccount `reader-sa`，只能 get/list pods，不能 delete
2. 建立 ServiceAccount `deploy-sa`，只能管 deployments
3. 用 `auth can-i` 驗證兩個 SA 的權限邊界
4. 找出 cluster 裡有哪些 SA 被綁定了 cluster-admin

**驗證：**
```bash
kubectl auth can-i delete pods -n default --as=system:serviceaccount:default:reader-sa
# 應該 no
kubectl auth can-i list pods -n default --as=system:serviceaccount:default:reader-sa
# 應該 yes
```

---

### 🔴 ServiceAccount 安全
**任務：**
1. 建立一個 SA，設定 `automountServiceAccountToken: false`
2. 建立使用這個 SA 的 pod，確認沒有 token 被 mount
3. 把 default namespace 的 default SA 也關掉 auto-mount
4. 確認 pod 內找不到 `/var/run/secrets/kubernetes.io/serviceaccount/token`

**驗證：**
```bash
kubectl exec <pod> -- ls /var/run/secrets/kubernetes.io/serviceaccount/
# 應該沒有 token 檔案
```

---

### 🟡 ClusterRole vs Role 範圍驗證
**任務：**
1. 建立 ClusterRole + RoleBinding（只在 dev namespace）
2. 建立 ClusterRole + ClusterRoleBinding（cluster-wide）
3. 驗證同一個 ClusterRole，不同 binding 類型，存取範圍不同

**驗證：**
```bash
kubectl auth can-i list pods -n dev --as=...    # yes
kubectl auth can-i list pods -n prod --as=...   # no（RoleBinding 只在 dev）
```

---

## 四、NetworkPolicy

### 🔴 Default Deny + 精準開放
**任務：**
1. 建三個 pod：frontend、backend、db（各自有 label）
2. 套用 default deny all
3. 只允許 frontend → backend（port 80）
4. 只允許 backend → db（port 5432）
5. 確認 frontend 無法直接連 db

**驗證：**
```bash
kubectl exec frontend -- wget -qO- --timeout=2 http://backend   # 成功
kubectl exec frontend -- wget -qO- --timeout=2 http://db        # 失敗
kubectl exec backend  -- wget -qO- --timeout=2 http://db        # 成功
```

---

### 🔴 AND vs OR selector 實驗
**任務：**
1. 建立一個用 AND（同一個 from 條目）的 NetworkPolicy
2. 建立一個用 OR（分開的 from 條目）的 NetworkPolicy
3. 各自用不同 pod 測試，確認行為不同

---

### 🔴 DNS Egress 驗證
**任務：**
1. 套用只有 Egress deny 的 NetworkPolicy（沒開 DNS）
2. 確認 pod 內 nslookup 失敗
3. 加上 UDP 53 egress rule
4. 確認 DNS 恢復

**驗證：**
```bash
kubectl exec <pod> -- nslookup kubernetes   # 加 DNS 前失敗，加後成功
```

---

### 🟡 跨 namespace 流量控制
**任務：**
1. 建立 namespace `frontend-ns` 和 `backend-ns`
2. 只允許 `frontend-ns` 的 pod 連到 `backend-ns` 的 pod
3. 驗證其他 namespace 被擋

---

## 五、Linux 加固

### 🔴 AppArmor
**任務：**
1. 確認 node 上 AppArmor 已啟用
2. 建立一個自訂 profile（限制寫入 /tmp 以外的路徑）
3. 載入 profile：`sudo apparmor_parser -q /etc/apparmor.d/my-profile`
4. 建立 pod 套用這個 profile（用 `securityContext.appArmorProfile`）
5. 在 pod 內嘗試寫入被限制的路徑，確認被擋

**驗證：**
```bash
sudo aa-status | grep my-profile   # profile 已載入
kubectl exec <pod> -- touch /etc/test-file   # 應該 Permission denied
```

---

### 🔴 Seccomp
**任務：**
1. 建立使用 `RuntimeDefault` seccomp 的 pod
2. 建立一個 localhost profile（限制特定 syscall）
3. 把 profile 放到 `/var/lib/kubelet/seccomp/profiles/`
4. 建立 pod 使用這個 localhost profile
5. 嘗試觸發被限制的 syscall

**驗證：**
```bash
kubectl get pod <pod> -o jsonpath='{.spec.securityContext.seccompProfile}'
```

---

### 🟡 Pod SecurityContext 完整硬化
**任務：**
1. 建立一個完整硬化的 pod：
   - `runAsNonRoot: true`
   - `readOnlyRootFilesystem: true`
   - `allowPrivilegeEscalation: false`
   - `capabilities.drop: [ALL]`
2. 用 emptyDir 掛載需要寫入的路徑
3. 確認 pod Running 且行為正常

**驗證：**
```bash
kubectl exec <pod> -- whoami          # 不應該是 root
kubectl exec <pod> -- touch /test     # Permission denied
kubectl exec <pod> -- touch /tmp/test # 成功（emptyDir）
```

---

## 六、Falco

### 🔴 安裝與觸發內建規則
**任務：**
1. 安裝 Falco（DaemonSet 或 systemd）
2. exec 進一個 running container 開 shell
3. 在 container 內讀取 `/etc/shadow`
4. 在 container 內執行 `curl`
5. 確認 Falco alert 有對應記錄

**驗證：**
```bash
journalctl -u falco --no-pager | tail -20
# 應該看到 shell 和 sensitive file 相關 alert
```

---

### 🔴 自訂規則
**任務：**
1. 寫一個規則：偵測 container 內執行 `wget` 或 `curl`
2. 寫一個規則：偵測 container 內讀取 `/etc/passwd`
3. 重啟 Falco，觸發這些行為
4. 確認 alert 格式包含 pod name、namespace、command

**驗證：**
```bash
sudo systemctl restart falco
kubectl exec <pod> -- curl example.com
journalctl -u falco | grep "curl"
```

---

## 七、ValidatingAdmissionPolicy

### 🔴 CEL 基本語法
**任務：**
1. 建立一個 policy：要求所有 pod 的 container 必須設定 `runAsNonRoot: true`
2. 建立 PolicyBinding 套用到 `production` namespace
3. 嘗試在 production 建立沒有 runAsNonRoot 的 pod，確認被擋
4. 建立符合條件的 pod，確認通過

**驗證：**
```bash
kubectl apply -f bad-pod.yaml -n production    # 應該失敗
kubectl apply -f good-pod.yaml -n production   # 應該成功
```

---

### 🔴 Registry 限制
**任務：**
1. 建立 policy：只允許 `docker.io/library/` 開頭的 image
2. 嘗試部署用其他 registry 的 pod，確認被擋

---

## 八、Pod Security Admission（PSA）

### 🔴 三個 level 實驗
**任務：**
1. 建立 namespace，套用 `enforce: baseline`
2. 嘗試部署 privileged pod，確認被擋
3. 換成 `enforce: restricted`
4. 嘗試部署沒有完整 SecurityContext 的 pod，確認被擋
5. 部署符合 restricted 的 pod，確認通過

**驗證：**
```bash
kubectl get ns <ns> --show-labels
kubectl apply -f privileged-pod.yaml -n <ns>   # 應該被擋
```

---

### 🟡 audit + warn mode
**任務：**
1. 套用 `audit: restricted`（不 enforce）
2. 建立不符合的 pod（成功），但 audit log 有記錄
3. 套用 `warn: restricted`，建立不符合的 pod，看到 warning 但不被擋

---

## 九、Supply Chain Security

### 🔴 Trivy Image Scanning
**任務：**
1. 掃描 `nginx:1.20`（舊版，有漏洞）
2. 只列出 CRITICAL severity
3. 掃描 `nginx:1.27-alpine`，比較漏洞數量
4. 掃描一個 Dockerfile

**驗證：**
```bash
trivy image --severity CRITICAL nginx:1.20
trivy image --severity CRITICAL nginx:1.27-alpine
# alpine 版本 CRITICAL 數量應明顯少很多
```

---

### 🟡 Dockerfile 最佳實踐
**任務：**
1. 寫一個「不好的」Dockerfile（FROM ubuntu、root user、COPY .）
2. 用 `trivy config Dockerfile` 掃描
3. 改成 multi-stage + non-root + alpine
4. 再掃一次比較

---

## 十、Ingress TLS（k3s 或 VM）

### 🟡 HTTPS Ingress
**任務：**
1. 用 openssl 產生自簽憑證
2. 建立 TLS secret
3. 建立 Ingress 設定 TLS
4. 確認 HTTPS 存取正常，HTTP redirect 到 HTTPS

**驗證：**
```bash
curl -k https://<hostname>   # 成功
curl http://<hostname>        # redirect 到 https
```

---

## 十一、Secrets 進階（k3s 或 VM）

### 🟡 Volume Mount vs Env Var
**任務：**
1. 建立 secret，用 env var 注入，確認 pod 內可以讀到
2. 改用 volume mount，確認以檔案形式存在
3. 更新 secret 的值，觀察兩種方式的更新行為差異

> volume mount 會在幾分鐘內自動更新，env var 不會（需要重啟 pod）

---

## 十二、PV / PVC / StorageClass

### 🟡 Reclaim Policy 驗證
**任務：**
1. 建立 PV，reclaim policy 設 `Retain`
2. 建立 PVC 綁定
3. 刪除 PVC，確認 PV 變成 Released 但資料還在
4. 換成 `Delete` policy，重複實驗，確認 PV 被刪除

---

## 十三、kube-bench（CIS Benchmark）

### 🔴 掃描與修復
**任務：**
1. 安裝 kube-bench，對 master 掃描
2. 找出 FAIL 的項目
3. 選 2-3 個修復（通常是 apiserver flags 或 kubelet config）
4. 重跑確認從 FAIL 變 PASS

**驗證：**
```bash
kube-bench run --targets=master | grep FAIL
kube-bench run --targets=master | grep -c PASS
```

---

## 練習順序建議

```
第一輪（打基礎）
  k8s 底層 → RBAC → NetworkPolicy → PSA → SecurityContext

第二輪（進階安全）
  Audit logging → API server 加固 → Falco → AppArmor/Seccomp

第三輪（考試導向）
  ValidatingAdmissionPolicy → Trivy → EncryptionConfig → kube-bench → killer.sh
```

---

## 環境對照

| 任務 | 環境 |
|------|------|
| etcd、kubeadm、kubelet、Static Pod、Audit logging、API server 加固 | Vagrant/VM |
| RBAC、NetworkPolicy、PSA、SecurityContext、Falco、Trivy、VAP | k3s |
| AppArmor、Seccomp | k3s 或 VM（需要 node 存取） |
| Ingress TLS、PV/PVC、Secrets | 兩者皆可 |
