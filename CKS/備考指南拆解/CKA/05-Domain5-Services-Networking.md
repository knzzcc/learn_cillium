# Domain 5 — Services & Networking（20%）

## Service 類型

| 類型 | 用途 |
|------|------|
| ClusterIP | cluster 內部溝通（預設） |
| NodePort | 從外部透過節點 IP 存取 |
| LoadBalancer | 雲端環境，取得外部 IP |

```bash
k expose deployment webapp --port=80 --target-port=8080 --type=ClusterIP
k expose deployment webapp --port=80 --target-port=8080 --type=NodePort
```

## NetworkPolicy

> ⚠️ 一旦對 Pod 套用任何 NetworkPolicy，預設全部 deny。沒有明確 allow 就進不來。

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: web          # 套用對象
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
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: db
    ports:
    - protocol: TCP
      port: 5432
  # DNS 一定要開，否則 DNS 解析失敗
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

### ingress from 的 AND vs OR 陷阱

```yaml
# AND：同時符合 namespace 且 pod label 才允許
from:
- namespaceSelector:
    matchLabels:
      env: prod
  podSelector:
    matchLabels:
      app: frontend

# OR：符合其中一個就允許
from:
- namespaceSelector:
    matchLabels:
      env: prod
- podSelector:
    matchLabels:
      app: frontend
```

## Ingress（傳統）

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
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

## Gateway API（v1.35 GA，考試重點）

```yaml
# Gateway
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
---
# HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /app
    backendRefs:
    - name: my-service
      port: 80
```

## DNS 除錯

```bash
# DNS 解析測試
k run test --image=busybox:1.36 --rm -it -- nslookup kubernetes
k run test --image=busybox:1.36 --rm -it -- nslookup my-svc.my-ns.svc.cluster.local

# CoreDNS 狀態
k get pods -n kube-system -l k8s-app=kube-dns
k logs -n kube-system -l k8s-app=kube-dns
```

DNS 規則：
- 同 namespace：`my-svc`
- 跨 namespace：`my-svc.my-ns`
- 完整：`my-svc.my-ns.svc.cluster.local`

## kubectl debug（GA）

```bash
# Debug Pod（建立 ephemeral container）
k debug pod/<pod-name> -it --image=busybox:1.36

# Debug Node（掛載節點 root fs）
k debug node/<node-name> -it --image=busybox:1.36
```
