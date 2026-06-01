# Domain 1 — Cluster Setup（15%）

## NetworkPolicy

一旦對 Pod 套用任何 NetworkPolicy，所有沒有明確 allow 的流量都會被 deny。

```yaml
# Default deny all（先套用這個，再逐一開放）
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# 允許特定流量 + DNS egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # DNS 必須開，否則 service name 解析失敗
  - to: []
    ports:
    - protocol: UDP
      port: 53
```

### AND vs OR 陷阱

```yaml
# AND：同時符合 namespace 且 pod label 才允許（同一個 - 下面）
from:
- namespaceSelector:
    matchLabels:
      env: prod
  podSelector:
    matchLabels:
      app: frontend

# OR：符合其中一個就允許（各自獨立的 -）
from:
- namespaceSelector:
    matchLabels:
      env: prod
- podSelector:
    matchLabels:
      app: frontend
```

### 封鎖 metadata API

```yaml
egress:
- to:
  - ipBlock:
      cidr: 0.0.0.0/0
      except:
      - 169.254.169.254/32   # 雲端 metadata API
```

## CIS Benchmark（kube-bench）

```bash
kube-bench run --targets=master
kube-bench run --targets=node
```

API server 常見加固 flags（`/etc/kubernetes/manifests/kube-apiserver.yaml`）：
```
--anonymous-auth=false
--authorization-mode=Node,RBAC
--enable-admission-plugins=NodeRestriction
--audit-log-path=/var/log/kubernetes/audit/audit.log
--profiling=false
--insecure-port=0
```

Kubelet 加固（`/var/lib/kubelet/config.yaml`）：
```yaml
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
authorization:
  mode: Webhook
readOnlyPort: 0
protectKernelDefaults: true
```

修改後：`sudo systemctl restart kubelet`

## Ingress TLS

```bash
# 產生憑證
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=app.example.com"

# 建立 TLS secret
k create secret tls tls-secret --cert=tls.crt --key=tls.key -n <ns>
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: tls-secret
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

---

# Domain 2 — Cluster Hardening（15%）

## RBAC 最小權限

```bash
# 找過度授權的帳號
k get clusterrolebinding -o wide | grep cluster-admin
k auth can-i --list --as=system:serviceaccount:default:default

# 建立最小權限 Role
k create role pod-reader --verb=get,list,watch --resource=pods -n dev
k create rolebinding read-pods --role=pod-reader \
  --serviceaccount=dev:my-sa -n dev

# 驗證
k auth can-i list pods -n dev --as=system:serviceaccount:dev:my-sa
k auth can-i delete pods -n dev --as=system:serviceaccount:dev:my-sa
```

## ServiceAccount 安全

```yaml
# 關閉 auto-mount（ServiceAccount 層級）
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secure-sa
automountServiceAccountToken: false
---
# 或在 Pod 層級關閉
spec:
  serviceAccountName: secure-sa
  automountServiceAccountToken: false
```

```bash
# 批量關閉 default SA 的 auto-mount
k patch sa default -n <ns> -p '{"automountServiceAccountToken": false}'

# 確認 SA 沒有多餘的 binding
k get rolebinding,clusterrolebinding --all-namespaces -o wide | grep default
```

最佳實踐：
- 不要用 `default` SA 給 workload，建立專屬 SA
- 除非 Pod 真的需要 API 存取，否則關閉 auto-mount
- 使用 TokenRequest API 取得短期 token，而非長期 secret
