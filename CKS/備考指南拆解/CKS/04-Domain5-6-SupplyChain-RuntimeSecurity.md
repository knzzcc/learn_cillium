# Domain 5 — Supply Chain Security（20%）

## Dockerfile 最佳實踐

```dockerfile
# 不好：完整 OS image
FROM ubuntu:22.04

# 較好：Alpine
FROM python:3.12-alpine

# 最好：distroless（無 shell、無 package manager）
FROM gcr.io/distroless/python3-debian12
```

Multi-stage build（隔離 build 與 runtime 依賴）：
```dockerfile
FROM python:3.12-alpine AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --target=/app/deps -r requirements.txt

FROM python:3.12-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=builder /app/deps /app/deps
COPY . .
USER appuser
EXPOSE 8080
ENTRYPOINT ["python", "app.py"]
```

原則：
- 不用 `:latest`，用具體版本號
- 不把 secret 複製進 image
- 用非 root USER
- `.dockerignore` 排除 `.git`、`*.env`、`*.key`

## Trivy 掃描

```bash
# 掃描 image
trivy image nginx:1.27
trivy image --severity CRITICAL,HIGH nginx:1.27
trivy image --severity CRITICAL nginx:1.27 --quiet
trivy image --severity CRITICAL nginx:1.27 -o results.txt

# 掃描 Dockerfile / config
trivy config Dockerfile
trivy config .
```

考試情境：掃描後找出 CRITICAL 漏洞，換成更安全的 image（通常是 alpine 版本）。

## 限制 Image Registry（ValidatingAdmissionPolicy）

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: restrict-registries
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
        c.image.startsWith('myregistry.io/') ||
        c.image.startsWith('docker.io/library/'))
    message: "Image 必須來自核准的 registry"
```

## 供應鏈安全重點

- 使用 image digest（`@sha256:...`）取代 tag，確保不可變
- 私有 registry 設定 imagePullSecrets
- 簽名驗證（cosign/Notary）
- 在 image 進入 cluster 前先掃描

---

# Domain 6 — Monitoring, Logging and Runtime Security（20%）

## Audit Logging

### Audit Policy

```yaml
# /etc/kubernetes/audit/policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# 不記錄 events
- level: None
  resources:
  - group: ""
    resources: ["events"]

# Secret 存取記錄 Metadata（不含 body）
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]

# Pod 異動記錄完整 request + response
- level: RequestResponse
  resources:
  - group: ""
    resources: ["pods"]

# 其餘記錄 Metadata
- level: Metadata
  omitStages:
  - RequestReceived
```

Audit level：
- `None`：不記錄
- `Metadata`：記錄 user、時間、resource、verb（不含 body）
- `Request`：Metadata + request body
- `RequestResponse`：Metadata + request body + response body

### API Server 設定

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml 加入
# flags:
#   --audit-log-path=/var/log/kubernetes/audit/audit.log
#   --audit-policy-file=/etc/kubernetes/audit/policy.yaml
#   --audit-log-maxage=30
#   --audit-log-maxbackup=10

# volumeMounts:
# - name: audit-policy
#   mountPath: /etc/kubernetes/audit
#   readOnly: true
# - name: audit-log
#   mountPath: /var/log/kubernetes/audit

# volumes:
# - name: audit-policy
#   hostPath:
#     path: /etc/kubernetes/audit
#     type: DirectoryOrCreate
# - name: audit-log
#   hostPath:
#     path: /var/log/kubernetes/audit
#     type: DirectoryOrCreate
```

> ⚠️ 修改 manifest 後 API server 會重啟，kubectl 可能 30-60 秒無回應，屬正常現象。

```bash
# 驗證 audit log 正在寫入
sudo tail -5 /var/log/kubernetes/audit/audit.log
sudo cat /var/log/kubernetes/audit/audit.log | jq 'select(.verb=="create")'
sudo cat /var/log/kubernetes/audit/audit.log | jq 'select(.user.username=="<user>")'
```

## Falco

### 安裝與管理

```bash
# 確認狀態
systemctl status falco
journalctl -u falco --no-pager | tail -30

# 查看 alert
cat /var/log/syslog | grep falco
journalctl -u falco -f

# 規則位置
ls /etc/falco/rules.d/
```

### 自訂規則

```yaml
# /etc/falco/rules.d/custom-rules.yaml
- rule: 偵測 container 內使用 curl 或 wget
  desc: 偵測可能的資料洩漏工具
  condition: >
    spawned_process and
    container and
    (proc.name in (curl, wget))
  output: >
    在 container 內偵測到 curl/wget
    (user=%user.name command=%proc.cmdline
    container=%container.name image=%container.image.repository
    namespace=%k8s.ns.name pod=%k8s.pod.name)
  priority: WARNING
  tags: [network, process, mitre_exfiltration]
```

```bash
# 修改規則後重啟
sudo systemctl restart falco

# 測試觸發
k exec -it <pod> -- curl example.com

# 確認 alert 出現
journalctl -u falco --no-pager | tail -10
```

### Falco 常見內建規則觸發條件

- Container 內啟動 shell（bash、sh）
- 讀取敏感檔案（`/etc/shadow`、`/etc/passwd`）
- 非預期的 process 啟動
- Namespace 切換（setns）
- 寫入系統目錄

## Container 不可變性

```yaml
# 確保 container 在 runtime 無法被修改
spec:
  containers:
  - name: app
    image: nginx:1.27
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
      runAsNonRoot: true
      runAsUser: 1000
    # 需要寫入的路徑用 emptyDir 掛載
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

## 實戰情境速查

### NetworkPolicy Default Deny + Allow + DNS

```yaml
# 1. deny all
# 2. allow-frontend-to-api（ingress on port 8080）
# 3. allow-dns（egress UDP 53 to anywhere）
# 詳見 Domain 1
```

### Audit Logging 設定流程

1. 寫 policy.yaml 到 `/etc/kubernetes/audit/`
2. 修改 kube-apiserver manifest 加 flags + volumeMounts + volumes
3. 等待 API server 重啟（30-60s）
4. 驗證 log 檔案存在且有內容

### Falco 自訂規則流程

1. 寫 `.yaml` 到 `/etc/falco/rules.d/`
2. `sudo systemctl restart falco`
3. 觸發測試行為
4. `journalctl -u falco` 確認 alert

### PSA 設定流程

1. `k label ns <ns> pod-security.kubernetes.io/enforce=restricted`
2. 建立符合 restricted 標準的 Pod（需要 securityContext 完整硬化）
3. `k get pod` 確認 Running
