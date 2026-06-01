# CKS YAML 範本速查

考試時需要能從記憶默寫的 YAML，按使用頻率排序。

---

## NetworkPolicy

### Default Deny All
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: <ns>
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Allow + DNS Egress
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app
  namespace: <ns>
spec:
  podSelector:
    matchLabels:
      app: <target>
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: <source>
    ports:
    - protocol: TCP
      port: <port>
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53
```

---

## SecurityContext（完整硬化）

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: <image>
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

---

## AppArmor（v1.34 field 寫法）

```yaml
containers:
- name: app
  image: <image>
  securityContext:
    appArmorProfile:
      type: Localhost
      localhostProfile: <profile-name>
    # type: RuntimeDefault（不需要 localhostProfile）
```

---

## Seccomp

```yaml
# Pod 層級
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault

# Localhost profile
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/my-profile.json
```

---

## Pod Security Admission（Namespace label）

```yaml
metadata:
  name: <ns>
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.34
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

---

## Audit Policy

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: None
  resources:
  - group: ""
    resources: ["events"]
- level: RequestResponse
  resources:
  - group: ""
    resources: ["pods"]
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]
- level: Metadata
  omitStages:
  - RequestReceived
```

---

## Encryption Config（靜態加密 Secrets）

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-32-bytes>
  - identity: {}   # 必須放最後
```

---

## Falco 自訂規則

```yaml
- rule: <rule-name>
  desc: <description>
  condition: >
    spawned_process and
    container and
    (proc.name in (curl, wget))
  output: >
    偵測到 (user=%user.name command=%proc.cmdline
    container=%container.name pod=%k8s.pod.name
    namespace=%k8s.ns.name)
  priority: WARNING
  tags: [process, network]
```

---

## Ingress TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <name>
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - <hostname>
    secretName: <tls-secret>
  rules:
  - host: <hostname>
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: <svc>
            port:
              number: 80
```

---

## ValidatingAdmissionPolicy

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: <name>
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["pods"]
  validations:
  - expression: "<CEL expression>"
    message: "<error message>"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: <name>-binding
spec:
  policyName: <name>
  validationActions:
  - Deny
  matchResources:
    namespaceSelector:
      matchLabels:
        <label-key>: <label-value>
```

---

## RuntimeClass

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
```

---

## ServiceAccount（關閉 auto-mount）

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <name>
  namespace: <ns>
automountServiceAccountToken: false
```
