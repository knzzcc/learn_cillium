# Domain 3 — System Hardening（10%）

## AppArmor（GA，v1.34 用 field，不是 annotation）

```bash
# 確認 profile 已載入
sudo aa-status
sudo apparmor_parser -q /etc/apparmor.d/my-profile
```

```yaml
# 正確寫法（v1.34 GA）
spec:
  containers:
  - name: app
    image: nginx:1.27
    securityContext:
      appArmorProfile:
        type: Localhost
        localhostProfile: my-custom-profile   # node 上已載入的 profile 名稱
```

```yaml
# 其他 type 選項
securityContext:
  appArmorProfile:
    type: RuntimeDefault   # container runtime 預設
    # type: Unconfined     # 不限制（避免使用）
```

> ⚠️ 舊版 annotation `container.apparmor.security.beta.kubernetes.io/<name>` 已廢棄，考試用 field 寫法。

## Seccomp

```yaml
# Pod 層級（推薦）
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault   # 最常用

# Localhost 自訂 profile
# profile 檔案路徑：/var/lib/kubelet/seccomp/profiles/my-profile.json
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/my-profile.json
```

## 縮小 Host OS 攻擊面

```bash
# 查看執行中的服務
systemctl list-units --type=service --state=running

# 關閉不必要的服務
sudo systemctl disable --now <service>

# 查看開放的 port
sudo ss -tlnp

# 找非必要的外部 service
k get svc --all-namespaces | grep -E "NodePort|LoadBalancer"
```

---

# Domain 4 — Minimize Microservice Vulnerabilities（20%）

## Pod SecurityContext 完整硬化範本

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:1.27
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
```

重要欄位說明：
- `runAsNonRoot: true`：禁止以 root 執行
- `readOnlyRootFilesystem: true`：root filesystem 唯讀（需要寫入的目錄用 emptyDir 掛載）
- `allowPrivilegeEscalation: false`：禁止提權
- `capabilities.drop: [ALL]`：移除所有 Linux capabilities

## Pod Security Admission（PSA）

三個 level：
- `privileged`：無限制
- `baseline`：防止已知提權方式
- `restricted`：最嚴格（最佳實踐）

三個 mode：
- `enforce`：違反直接拒絕
- `audit`：記錄但放行
- `warn`：警告但放行

```bash
# 套用到 namespace
k label ns secure-ns pod-security.kubernetes.io/enforce=restricted
k label ns secure-ns pod-security.kubernetes.io/audit=restricted
k label ns secure-ns pod-security.kubernetes.io/warn=restricted
```

```yaml
# 或在 Namespace manifest 寫
metadata:
  name: secure-ns
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.34
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

## Secrets 管理

```bash
# 建立
k create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=s3cret

# 解碼查看
k get secret db-creds -o jsonpath='{.data.password}' | base64 -d
```

```yaml
# 注入方式 1：環境變數（單一 key）
env:
- name: DB_USER
  valueFrom:
    secretKeyRef:
      name: db-creds
      key: username

# 注入方式 2：Volume mount（較安全，以檔案形式）
volumeMounts:
- name: secret-vol
  mountPath: /etc/db-creds
  readOnly: true
volumes:
- name: secret-vol
  secret:
    secretName: db-creds
```

### 加密靜態 Secrets

```bash
# 產生加密 key
head -c 32 /dev/urandom | base64
```

```yaml
# /etc/kubernetes/enc/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:          # 第一個 = 用於加密
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
  - identity: {}     # 最後一個 = 解密 fallback，不可放第一
```

```bash
# 在 kube-apiserver manifest 加入
# --encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml
# 並加入對應 volumeMount

# 重新加密現有 secrets
k get secrets --all-namespaces -o json | k replace -f -
```

## ValidatingAdmissionPolicy（CEL）

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-non-root
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["pods"]
  validations:
  - expression: >
      object.spec.containers.all(c,
        has(c.securityContext) &&
        has(c.securityContext.runAsNonRoot) &&
        c.securityContext.runAsNonRoot == true)
    message: "所有 container 必須設定 runAsNonRoot: true"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: require-non-root-binding
spec:
  policyName: require-non-root
  validationActions:
  - Deny
  matchResources:
    namespaceSelector:
      matchLabels:
        env: production
```

## RuntimeClass（gVisor / Kata）

```yaml
# 建立 RuntimeClass
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
---
# Pod 使用 sandbox runtime
spec:
  runtimeClassName: gvisor
  containers:
  - name: app
    image: nginx:1.27
```
