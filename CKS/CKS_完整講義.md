# CKS 認證考試完整講義
**Certified Kubernetes Security Specialist**

> 版本：Kubernetes v1.29+
> 考試時間：120 分鐘 | 及格分數：67%
> 考試形式：實作操作（Performance-Based）

---

## 目錄

1. [Cluster Setup & Hardening](#1-cluster-setup--hardening)
2. [System Hardening](#2-system-hardening)
3. [Minimize Microservice Vulnerabilities](#3-minimize-microservice-vulnerabilities)
4. [Supply Chain Security](#4-supply-chain-security)
5. [Monitoring, Logging & Runtime Security](#5-monitoring-logging--runtime-security)
6. [常用指令速查表](#6-常用指令速查表)

---

## 1. Cluster Setup & Hardening

> 考試佔比約 **15%**

### 1.1 CIS Benchmarks 概念

#### 什麼是 CIS Benchmark？

CIS（Center for Internet Security）是一個非營利組織，針對各種系統制定安全設定標準。**CIS Kubernetes Benchmark** 是一份文件，列出了 Kubernetes 叢集各元件的安全設定建議，分為以下幾個檢查類別：

- **Master Node 安全設定**：API Server、Scheduler、Controller Manager、etcd
- **Worker Node 安全設定**：kubelet 設定
- **Policies**：RBAC、Pod Security、Network Policy

#### 為什麼重要？

剛建立好的 Kubernetes 叢集預設並不安全，例如：
- API Server 可能允許匿名存取
- 未啟用 Audit Logging
- etcd 中的資料未加密

CIS Benchmark 提供了一個「安全檢查清單」，讓你知道哪些地方需要修正。

#### 評分標準

CIS 將每項設定分為兩種：
- **Scored**：必須達成，影響合規分數
- **Not Scored**：建議達成，但不影響分數

#### 如何掃描？

工具 **kube-bench** 可以自動對照 CIS Benchmark 掃描叢集：

```bash
# 以 Job 形式在叢集內執行
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs job/kube-bench

# 針對特定角色執行
kube-bench run --targets master
kube-bench run --targets node
kube-bench run --targets etcd
```

掃描結果會顯示 `[PASS]`、`[FAIL]`、`[WARN]` 三種狀態，並附上修正建議。

---

### 1.2 Kube-apiserver 安全設定

#### API Server 的角色

API Server 是 Kubernetes 的「大門」，所有對叢集的操作（kubectl、Controller、kubelet）都必須經過它。因此，API Server 的安全設定直接決定了叢集的安全邊界。

API Server 的請求處理流程：

```
請求進入
   ↓
Authentication（認證）：你是誰？
   ↓
Authorization（授權）：你有沒有權限做這件事？
   ↓
Admission Control（准入控制）：這個請求符合政策嗎？
   ↓
寫入 etcd / 執行操作
```

#### 重要安全參數說明

| 參數 | 說明 | 安全建議 |
|------|------|---------|
| `--anonymous-auth` | 是否允許匿名存取 | 設為 `false` |
| `--authorization-mode` | 授權模式 | 使用 `Node,RBAC`（不可只用 `AlwaysAllow`） |
| `--insecure-port` | HTTP 明文 port | 設為 `0`（停用） |
| `--enable-admission-plugins` | 啟用的 Admission Controller | 包含 `NodeRestriction` |
| `--audit-log-path` | Audit log 輸出路徑 | 必須設定 |
| `--tls-min-version` | 最低 TLS 版本 | `VersionTLS12` 以上 |

#### 設定方式

kube-apiserver 以靜態 Pod 形式運行，設定檔位於：

```
/etc/kubernetes/manifests/kube-apiserver.yaml
```

修改後 kubelet 會自動重啟 Pod（約需 1-2 分鐘），無需手動操作。

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --anonymous-auth=false
    - --authorization-mode=Node,RBAC
    - --enable-admission-plugins=NodeRestriction,PodSecurityAdmission
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-log-maxage=30
    - --audit-log-maxbackup=10
    - --audit-log-maxsize=100
    - --tls-min-version=VersionTLS12
    - --insecure-port=0
```

---

### 1.3 RBAC（角色型存取控制）

#### 概念：最小權限原則

在 Kubernetes 中，每個使用者、服務帳號（ServiceAccount）預設應該「什麼都不能做」，只有明確授予的權限才能使用。這就是**最小權限原則（Principle of Least Privilege）**。

RBAC 就是實現這個原則的機制。

#### 核心物件與關係

RBAC 由四種物件組成，關係如下：

```
使用者 / ServiceAccount
        ↓ 透過
  RoleBinding / ClusterRoleBinding
        ↓ 參照
     Role / ClusterRole
        ↓ 定義
    可對哪些資源做哪些操作
```

| 物件 | 作用範圍 | 說明 |
|------|---------|------|
| `Role` | Namespace | 定義某個 namespace 內的權限 |
| `ClusterRole` | 全叢集 | 定義叢集層級的權限（可跨 namespace） |
| `RoleBinding` | Namespace | 將 Role 或 ClusterRole 指派給某人（在 namespace 內生效） |
| `ClusterRoleBinding` | 全叢集 | 將 ClusterRole 指派給某人（在全叢集生效） |

> **注意**：ClusterRole 可以被 RoleBinding 參照，效果是「在該 namespace 內套用 ClusterRole 定義的權限」，不會擴展到全叢集。

#### verbs 與 resources

```yaml
rules:
- apiGroups: [""]        # "" 代表 core API group（pods, services 等）
  resources: ["pods"]    # 哪種資源
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

常用 verbs：

| Verb | 說明 |
|------|------|
| `get` | 取得單一資源 |
| `list` | 列出資源（不含內容） |
| `watch` | 監聽資源變化 |
| `create` | 建立資源 |
| `update` | 完整更新資源 |
| `patch` | 部分更新資源 |
| `delete` | 刪除資源 |
| `*` | 所有操作 |

#### 實作範例

```yaml
# 1. 建立 Role：允許讀取 pods
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

---
# 2. 建立 RoleBinding：將 Role 指派給 jane
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

#### 驗證權限

```bash
# 確認自己的權限
kubectl auth can-i get pods

# 模擬其他使用者的權限
kubectl auth can-i get pods --as=jane
kubectl auth can-i get pods --as=system:serviceaccount:default:my-sa

# 列出某個 namespace 下所有可做的事
kubectl auth can-i --list -n default
```

---

### 1.4 ServiceAccount 安全

#### 什麼是 ServiceAccount？

ServiceAccount 是讓**程式（Pod）**與 Kubernetes API 溝通時使用的身份。每個 Pod 預設會自動掛載一個 token，讓 Pod 內的程式可以呼叫 API Server。

問題在於：大多數應用程式根本不需要存取 Kubernetes API，但預設的自動掛載行為讓所有 Pod 都帶著 token，增加了被攻擊後的風險。

#### 安全建議

1. **停用不必要的 token 自動掛載**
2. **遵循最小權限原則**：只給 ServiceAccount 必要的 RBAC 權限

```yaml
# 在 ServiceAccount 層級停用（影響使用此 SA 的所有 Pod）
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-sa
automountServiceAccountToken: false

---
# 在 Pod 層級停用（覆蓋 SA 的設定）
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  automountServiceAccountToken: false
  serviceAccountName: my-sa
  containers:
  - name: app
    image: nginx
```

---

### 1.5 Network Policy（網路政策）

#### 概念：Kubernetes 網路的預設行為

Kubernetes 預設允許所有 Pod 之間互相通訊，這對安全來說是很大的風險——一旦某個 Pod 被攻擊，攻擊者可以自由地探索並攻擊叢集內的其他服務。

Network Policy 讓你可以定義「哪些 Pod 可以和哪些 Pod 通訊」，就像防火牆規則一樣。

> **重要前提**：Network Policy 需要 CNI 插件支援（如 Calico、Cilium、Weave），使用 Flannel 等不支援 NetworkPolicy 的 CNI 時，規則會被忽略。

#### 運作邏輯

- **沒有任何 NetworkPolicy 選取某個 Pod 時**：該 Pod 的流量不受限制（預設全開放）
- **有 NetworkPolicy 選取某個 Pod 時**：只有規則中明確允許的流量才能通過，其餘全部拒絕

因此，安全實踐是先建立「預設拒絕所有」的規則，再逐一開放必要的流量。

#### podSelector 的含義

```yaml
spec:
  podSelector: {}        # 空的 selector = 選取此 namespace 中所有 Pod
  podSelector:
    matchLabels:
      app: backend       # 只選取有 app=backend 標籤的 Pod
```

#### 實作範例

```yaml
# 第一步：拒絕所有 ingress 流量（namespace 內所有 Pod）
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress

---
# 第二步：只允許 frontend 連到 backend 的 8080 port
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend        # 這個規則保護 backend Pod
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend   # 只允許來自 frontend Pod 的流量
    ports:
    - protocol: TCP
      port: 8080
```

#### 跨 Namespace 的流量控制

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        name: monitoring     # 允許來自 monitoring namespace 的流量
    podSelector:
      matchLabels:
        app: prometheus      # 且必須是帶有此標籤的 Pod
```

> **注意**：`namespaceSelector` 和 `podSelector` 在同一個 `-` 項目下是 AND 關係；分開成兩個 `-` 項目則是 OR 關係。

---

### 1.6 Etcd 安全與資料加密

#### Etcd 的重要性

etcd 是 Kubernetes 的資料庫，儲存了叢集的所有狀態，包括 Secrets。問題是，etcd 預設**不加密**儲存 Secrets，如果有人能直接存取 etcd，就能讀取所有 Secrets 的明文內容。

#### 靜態加密（Encryption at Rest）

Kubernetes 支援在寫入 etcd 前先加密資料，稱為 Encryption at Rest。

**加密流程：**

```
kubectl create secret → API Server → 加密（AES-CBC / AES-GCM）→ 寫入 etcd
讀取 secret → API Server ← 解密 ← 從 etcd 讀取
```

**設定步驟：**

1. 建立加密設定檔：

```yaml
# /etc/kubernetes/enc/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets                  # 要加密的資源種類
  providers:
  - aescbc:                  # 加密演算法（寫入時用第一個 provider）
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>   # echo -n "1234567890123456789012345678901234" | base64
  - identity: {}             # identity = 不加密（放在後面表示可讀取舊的未加密資料）
```

2. 在 kube-apiserver 指定設定檔：

```yaml
- --encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml
```

3. 重新加密現有 Secrets（讓舊資料也套用加密）：

```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

4. 驗證加密是否生效（應看到 `k8s:enc:aescbc:v1:` 開頭）：

```bash
ETCDCTL_API=3 etcdctl get /registry/secrets/default/my-secret \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt | hexdump -C
```

---

## 2. System Hardening

> 考試佔比約 **15%**

### 2.1 最小化 OS 攻擊面

#### 攻擊面（Attack Surface）的概念

攻擊面是指系統中所有可能被攻擊者利用的入口點。一個運行了不必要服務的節點，就算 Kubernetes 設定得再好，也可能因為 OS 層面的漏洞而被入侵。

**減少攻擊面的原則：**
- 只安裝必要的套件
- 只開放必要的 port
- 只啟動必要的服務
- 使用者帳號遵循最小權限

```bash
# 查看正在執行的服務
systemctl list-units --type=service --state=running

# 停用不需要的服務
systemctl disable --now snapd

# 查看開放的 port 和對應程序
ss -tlnp
netstat -tlnp

# 移除不需要的套件
apt list --installed
apt purge <package-name>
```

---

### 2.2 AppArmor

#### 什麼是 AppArmor？

AppArmor（Application Armor）是 Linux 核心的安全模組，屬於 MAC（Mandatory Access Control，強制存取控制）機制。它讓你可以為每個程式定義一個「白名單」，限制該程式能存取哪些檔案、網路、系統呼叫等。

即使容器內的程式被攻擊者控制，AppArmor 也能防止它做出超出預期的行為。

#### 兩種模式

| 模式 | 說明 |
|------|------|
| `enforce` | 強制執行 profile，違規的操作會被拒絕並記錄 |
| `complain` | 只記錄違規操作，不實際拒絕（用於測試 profile） |

#### Profile 的結構

```
#include <tunables/global>

profile 名稱 flags=(attach_disconnected) {
  #include <abstractions/base>

  # 允許讀取所有檔案
  file,

  # 拒絕所有寫入操作（deny 優先於 allow）
  deny /** w,

  # 允許網路連線
  network tcp,
}
```

#### 操作步驟

```bash
# 1. 確認節點已載入 AppArmor
aa-status

# 2. 將 profile 檔案放到 /etc/apparmor.d/
# 例如：/etc/apparmor.d/k8s-deny-write

# 3. 載入 profile
apparmor_parser -q /etc/apparmor.d/k8s-deny-write          # enforce 模式
apparmor_parser -C /etc/apparmor.d/k8s-deny-write           # complain 模式

# 4. 確認 profile 已載入
aa-status | grep k8s-deny-write
```

#### 在 Pod 中套用

AppArmor 透過 Pod annotation 指定（格式：`container.apparmor.security.beta.kubernetes.io/<容器名稱>: localhost/<profile名稱>`）：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-pod
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/k8s-deny-write
spec:
  containers:
  - name: app
    image: nginx
```

> **注意**：profile 必須預先載入到 **Pod 所在的節點**上，否則 Pod 會無法啟動。

---

### 2.3 Seccomp

#### 什麼是 Seccomp？

Seccomp（Secure Computing Mode）是 Linux 核心的功能，可以限制程序能呼叫哪些**系統呼叫（syscall）**。

容器內的程式最終都是透過 syscall 與 Linux 核心溝通（例如：open 開檔、socket 建立網路連線、execve 執行程式）。攻擊者在取得容器控制權後，往往需要特定的 syscall 來提權或逃逸，Seccomp 能有效阻止這些行為。

#### Seccomp Profile 類型

| 類型 | 說明 |
|------|------|
| `Unconfined` | 不限制（預設） |
| `RuntimeDefault` | 使用容器 runtime 的預設 profile（較安全，推薦） |
| `Localhost` | 使用節點上自訂的 profile 檔案 |

#### Profile 結構

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",   // 預設動作：拒絕並回傳錯誤
  "syscalls": [
    {
      "names": ["read", "write", "open", "close", "stat", "fstat"],
      "action": "SCMP_ACT_ALLOW"        // 允許這些 syscall
    }
  ]
}
```

常用 defaultAction：

| Action | 說明 |
|--------|------|
| `SCMP_ACT_ALLOW` | 允許 |
| `SCMP_ACT_ERRNO` | 拒絕，回傳錯誤碼 |
| `SCMP_ACT_LOG` | 允許但記錄（用於審計/測試） |
| `SCMP_ACT_KILL` | 直接終止程序 |

#### 在 Pod 中套用

Profile 檔案需放在節點的 `/var/lib/kubelet/seccomp/` 目錄下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-pod
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault        # 使用 runtime 預設 profile
  containers:
  - name: app
    image: nginx

---
# 使用自訂 profile（/var/lib/kubelet/seccomp/profiles/audit.json）
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json
```

---

### 2.4 Linux Capabilities

#### 什麼是 Capabilities？

傳統 Linux 將權限分為兩種：root（全部權限）和非 root（幾乎沒有特殊權限）。這種設計過於粗糙，因此 Linux 引入了 **Capabilities**，將 root 的特殊權限拆分成數十個獨立的細項。

常見的 Capabilities：

| Capability | 說明 |
|-----------|------|
| `NET_BIND_SERVICE` | 允許綁定 1024 以下的 port |
| `NET_ADMIN` | 允許各種網路操作（修改路由、防火牆等） |
| `SYS_ADMIN` | 非常廣泛的系統管理權限（高危！） |
| `SYS_PTRACE` | 允許 ptrace 系統呼叫（可用於偵錯和攻擊） |
| `CHOWN` | 允許變更檔案擁有者 |
| `SETUID` / `SETGID` | 允許切換使用者/群組 |

容器預設會帶有一些 Capabilities，即使不是以 root 執行。最佳實踐是**先移除所有 Capabilities，再只加回真正需要的**。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: capability-demo
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      capabilities:
        drop:
        - ALL                    # 先移除所有 capabilities
        add:
        - NET_BIND_SERVICE       # 只加回 nginx 需要的
```

---

### 2.5 Pod Security Standards (PSS)

#### 背景：PodSecurityPolicy 的替代方案

舊版 Kubernetes 使用 PodSecurityPolicy（PSP）來限制 Pod 的安全設定，但 PSP 操作複雜且易出錯，已在 v1.25 被移除。取而代之的是 **Pod Security Standards（PSS）**，透過 **Pod Security Admission（PSA）** Controller 實作。

#### 三種安全等級

| 等級 | 說明 | 適用場景 |
|------|------|---------|
| `privileged` | 完全不限制，允許特權操作 | 系統元件、需要特殊權限的工作負載 |
| `baseline` | 防止已知的特權提升手法，影響最小 | 一般應用程式 |
| `restricted` | 遵循最嚴格的安全最佳實踐 | 安全敏感的應用程式 |

#### 三種操作模式

| 模式 | 行為 |
|------|------|
| `enforce` | 違規的 Pod 直接被拒絕建立 |
| `audit` | 違規時記錄到 audit log，但允許建立 |
| `warn` | 違規時顯示警告給使用者，但允許建立 |

#### 在 Namespace 套用

透過 label 設定，可以組合使用不同模式和等級：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-apps
  labels:
    # 強制執行 baseline（不符合就拒絕）
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: latest
    # 審計 restricted 等級的違規
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    # 對 restricted 等級的違規顯示警告
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

---

## 3. Minimize Microservice Vulnerabilities

> 考試佔比約 **20%**

### 3.1 SecurityContext

#### 概念

SecurityContext 讓你在 Pod 或容器層級設定安全相關的選項，是實作「最小權限容器」的核心工具。

設定可以在兩個層級：
- **Pod 層級**（`spec.securityContext`）：影響所有容器
- **容器層級**（`spec.containers[].securityContext`）：只影響特定容器，可覆蓋 Pod 層級的設定

#### 重要設定項目說明

| 設定 | 說明 | 建議值 |
|------|------|-------|
| `runAsNonRoot` | 拒絕以 root 執行的容器 | `true` |
| `runAsUser` | 指定執行的 UID | 非 0 的值，如 `1000` |
| `runAsGroup` | 指定執行的 GID | 非 0 的值 |
| `fsGroup` | 掛載 volume 時的群組 | 指定群組 ID |
| `allowPrivilegeEscalation` | 是否允許程序取得比父程序更高的權限（如 setuid） | `false` |
| `readOnlyRootFilesystem` | 根目錄唯讀，防止攻擊者在容器內寫入檔案 | `true` |
| `privileged` | 特權容器（幾乎等同於 root 存取節點） | `false` |

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
  containers:
  - name: app
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp-dir
      mountPath: /tmp                  # 唯讀根目錄時，需要寫入的目錄另外掛載
  volumes:
  - name: tmp-dir
    emptyDir: {}
```

---

### 3.2 Open Policy Agent (OPA) / Gatekeeper

#### 概念

Kubernetes 的 Admission Controller 允許你在資源被建立/修改前插入自訂邏輯。OPA Gatekeeper 利用這個機制，讓你用 **Rego 語言**撰寫策略，統一管理叢集中的合規規則。

**兩種 Webhook 類型：**

| 類型 | 說明 |
|------|------|
| MutatingAdmissionWebhook | 可以修改請求（例如自動加上 label） |
| ValidatingAdmissionWebhook | 只能允許或拒絕請求（不能修改） |

OPA Gatekeeper 主要使用 **Validating** Webhook。

#### 架構

Gatekeeper 引入兩種 CRD：

1. **ConstraintTemplate**：定義「政策模板」，包含 Rego 邏輯（類似 Class）
2. **Constraint**：從模板建立的「政策實例」，指定套用範圍和參數（類似 Object）

```
ConstraintTemplate（定義規則邏輯）
         ↓ 產生
   CustomResourceDefinition（新的 K8s 資源類型）
         ↓ 建立
  Constraint（指定規則套用到哪些資源）
```

#### 實作範例：要求 Pod 必須有特定 label

```yaml
# 1. ConstraintTemplate：定義邏輯
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels

      violation[{"msg": msg}] {
        # 取得所有 label key
        provided := {label | input.review.object.metadata.labels[label]}
        # 取得要求的 label
        required := {label | label := input.parameters.labels[_]}
        # 找出缺少的 label
        missing := required - provided
        count(missing) > 0
        msg := sprintf("缺少必要的 labels: %v", [missing])
      }

---
# 2. Constraint：套用規則
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: pod-must-have-team
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters:
    labels: ["team", "env"]      # 要求 Pod 必須有這兩個 label
```

---

### 3.3 Secrets 管理最佳實踐

#### Secret 的本質與風險

Kubernetes Secret 預設只是 Base64 編碼，**不是加密**。Base64 可以直接解碼，因此：
- 有 `get secrets` 權限的人可以讀取所有 Secret 內容
- etcd 中的 Secret 如果未啟用加密，直接存取 etcd 就能看到明文

#### 安全使用原則

1. **以 Volume 掛載，優於環境變數**：環境變數容易被 `env` 指令或 log 洩漏
2. **啟用 etcd 加密**（見 1.6 節）
3. **使用 RBAC 限制誰能讀取 Secret**
4. **考慮使用外部 Secret 管理工具**（如 HashiCorp Vault、AWS Secrets Manager）

```yaml
# 推薦方式：以 Volume 掛載
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  volumes:
  - name: secret-vol
    secret:
      secretName: my-secret
      defaultMode: 0400        # 只有擁有者可讀，其他人無法存取
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secrets
      readOnly: true           # 以唯讀方式掛載
```

---

### 3.4 mTLS 與服務間通訊安全

#### 概念：為什麼需要 mTLS？

普通的 TLS 只驗證**伺服器**的身份（像是 HTTPS），但在微服務架構中，我們還需要驗證**呼叫方**的身份。**mTLS（Mutual TLS）**要求雙方都提供憑證，確保「我知道我在和誰說話，對方也知道在和我說話」。

在 Kubernetes 中，Service Mesh（如 Istio、Linkerd）可以自動為 Pod 間的通訊建立 mTLS，無需應用程式自行實作。

#### Istio mTLS 設定

```yaml
# 設定 namespace 內強制使用 mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT      # STRICT = 強制 mTLS；PERMISSIVE = 允許 mTLS 或明文

---
# 設定哪些服務可以存取 backend
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: backend-access
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
  - from:
    - source:
        # 只允許帶有此 ServiceAccount 憑證的呼叫方
        principals: ["cluster.local/ns/production/sa/frontend"]
    to:
    - operation:
        methods: ["GET", "POST"]
```

---

### 3.5 容器隔離與 RuntimeClass

#### 概念：不同程度的容器隔離

標準 Docker/containerd 容器共享主機核心，這帶來了隔離性的疑慮。對於高安全需求的工作負載，可以選擇更強的隔離機制：

| 隔離方式 | 工具 | 說明 |
|---------|------|------|
| 標準容器 | runc | 共享主機核心，隔離依賴 namespace/cgroups |
| 沙箱容器 | gVisor（runsc） | 在應用程式和核心之間加一層用戶空間核心 |
| VM 級隔離 | Kata Containers | 每個容器運行在輕量 VM 中，擁有獨立核心 |

#### 使用 RuntimeClass

```yaml
# 定義 RuntimeClass（通常由管理員建立）
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc          # 對應 containerd 或 cri-o 的 runtime handler 名稱

---
# Pod 使用 RuntimeClass
apiVersion: v1
kind: Pod
metadata:
  name: sandboxed-app
spec:
  runtimeClassName: gvisor   # 指定使用 gVisor
  containers:
  - name: app
    image: nginx
```

---

## 4. Supply Chain Security

> 考試佔比約 **20%**

### 4.1 概念：什麼是軟體供應鏈攻擊？

軟體供應鏈攻擊是指攻擊者在軟體的**生產、封裝、分發**過程中植入惡意程式碼，而非直接攻擊最終用戶。

在 Kubernetes 的情境下，供應鏈風險包括：
- 使用含有漏洞或後門的基底映像
- 從不受信任的 Registry 拉取映像
- CI/CD pipeline 被入侵，產生被篡改的映像
- 使用未經驗證的第三方 Helm Chart 或 YAML

**防禦策略：**
1. 掃描映像漏洞
2. 對映像進行簽署與驗證
3. 使用准入控制只允許受信任的映像
4. 最小化基底映像的攻擊面

---

### 4.2 映像漏洞掃描（Trivy）

#### Trivy 的工作原理

Trivy 會解析映像的每一層，取得已安裝的套件清單，再對照 CVE（Common Vulnerabilities and Exposures）資料庫，找出已知漏洞。

漏洞嚴重程度分為：CRITICAL > HIGH > MEDIUM > LOW > UNKNOWN

```bash
# 掃描映像
trivy image nginx:latest

# 只顯示高危以上漏洞
trivy image --severity HIGH,CRITICAL nginx:latest

# 掃描 Kubernetes YAML 設定檔（找設定錯誤）
trivy config ./manifests/

# 掃描整個叢集
trivy k8s --report summary cluster

# 在 CI/CD 中使用（找到 CRITICAL 漏洞時回傳非 0 exit code，讓 pipeline 失敗）
trivy image --exit-code 1 --severity CRITICAL my-app:v1.0
```

---

### 4.3 映像簽署與驗證（Cosign）

#### 概念：如何信任一個映像？

Tag（如 `nginx:latest`）可以被覆蓋，你無法確認拉下來的映像是否是原作者發布的。

**映像簽署**的流程：
1. 建置映像後，用**私鑰**對映像的 digest（SHA256）進行簽署，產生簽章
2. 將簽章推送到 Registry（或 Sigstore 透明日誌）
3. 部署前，用**公鑰**驗證映像的簽章，確認映像未被篡改且來自受信任的來源

```bash
# 生成金鑰對
cosign generate-key-pair
# 產生 cosign.key（私鑰）和 cosign.pub（公鑰）

# 簽署映像（推送後執行）
cosign sign --key cosign.key myregistry/myapp:v1.0

# 驗證簽章
cosign verify --key cosign.pub myregistry/myapp:v1.0
```

---

### 4.4 准入控制（Admission Controllers）

#### 概念：請求的最後防線

Admission Controller 是 Kubernetes 請求處理流程的最後一關，在資源寫入 etcd 之前執行。它可以：
- **拒絕**不符合政策的請求
- **修改**請求內容（如自動注入 sidecar）

#### 重要的 Admission Controllers

| 名稱 | 說明 |
|------|------|
| `NodeRestriction` | 限制 kubelet 只能修改自己節點的 Node 和 Pod 物件，防止節點橫向移動 |
| `PodSecurityAdmission` | 強制套用 Pod Security Standards（PSS） |
| `LimitRanger` | 為沒有設定資源限制的 Pod 自動加上預設值 |
| `ResourceQuota` | 限制 namespace 的總資源用量 |
| `AlwaysPullImages` | 強制每次都從 Registry 拉取最新映像（防止使用本地快取的惡意映像） |

```bash
# 查看 kube-apiserver 啟用的 admission plugins
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep admission
```

---

### 4.5 最小化基底映像

#### 為什麼要使用精簡映像？

映像中的每個套件都是潛在的攻擊面。如果 nginx 容器中還安裝了 curl、bash、python，攻擊者就有更多工具可以利用。

| 映像類型 | 說明 | 攻擊面 |
|---------|------|-------|
| `ubuntu:latest` | 完整 Ubuntu，包含大量工具 | 大 |
| `alpine:latest` | 極簡 Linux，約 5MB | 中 |
| `gcr.io/distroless/static` | 無 shell、無套件管理器，只有應用程式 | 極小 |
| `scratch` | 完全空白，只有應用程式二進位檔 | 最小 |

#### 多階段建置（Multi-stage Build）

```dockerfile
# 第一階段：使用完整環境建置應用程式
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server .

# 第二階段：只複製最終的二進位檔到精簡映像
FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app/server /server
USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

這樣最終映像中不會有 Go 編譯器、source code，攻擊者也無法在容器內執行 shell。

---

## 5. Monitoring, Logging & Runtime Security

> 考試佔比約 **20%**

### 5.1 Audit Logging

#### 概念：為什麼需要 Audit Log？

安全事故發生時，你需要回答：「誰、在什麼時候、對什麼資源、做了什麼操作？」Kubernetes Audit Log 記錄了所有對 API Server 的請求，提供完整的操作軌跡。

#### Audit Log 的四個處理階段

每個請求在不同處理階段都可以觸發 audit 記錄：

| 階段 | 說明 |
|------|------|
| `RequestReceived` | API Server 收到請求（尚未處理） |
| `ResponseStarted` | 開始回應（用於 long-running 請求如 watch） |
| `ResponseComplete` | 回應完成 |
| `Panic` | 發生 panic 時 |

#### 四種記錄等級（Level）

| Level | 記錄內容 |
|-------|---------|
| `None` | 不記錄此規則匹配的請求 |
| `Metadata` | 只記錄請求的 metadata（時間、用戶、資源、操作），**不含 request/response body** |
| `Request` | Metadata + request body |
| `RequestResponse` | Metadata + request body + response body（資料最完整，也最大） |

#### 撰寫 Audit Policy

規則從上到下匹配，**第一個匹配的規則生效**，後續規則不再評估。

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
- "RequestReceived"      # 忽略所有請求的 RequestReceived 階段（減少噪音）

rules:
# 規則 1：secrets 的所有操作只記錄 metadata（避免 secret 內容出現在 log）
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]

# 規則 2：pods 的 create/update/delete 記錄完整 request/response
- level: RequestResponse
  verbs: ["create", "update", "delete", "patch"]
  resources:
  - group: ""
    resources: ["pods"]

# 規則 3：系統元件的 watch/get 操作不記錄（避免 log 爆炸）
- level: None
  users:
  - "system:kube-proxy"
  - "system:node"
  verbs: ["get", "watch", "list"]

# 規則 4：其餘所有請求記錄 metadata
- level: Metadata
```

#### 在 kube-apiserver 啟用

```yaml
- --audit-log-path=/var/log/kubernetes/audit.log
- --audit-policy-file=/etc/kubernetes/audit-policy.yaml
- --audit-log-maxage=30          # 保留最近 30 天
- --audit-log-maxbackup=10       # 最多保留 10 個備份
- --audit-log-maxsize=100        # 每個 log 檔最大 100MB
```

---

### 5.2 Falco：執行期威脅偵測

#### 概念：靜態防護的不足

前面介紹的各種安全措施（NetworkPolicy、Seccomp、AppArmor 等）都是**靜態的預防**措施。但如果攻擊者找到了漏洞繞過這些防護，我們如何知道？

Falco 提供**動態偵測**能力，它即時監控系統呼叫和 Kubernetes API 事件，當發現可疑行為時立即告警。例如：
- 容器內有人開啟了 shell
- 有人讀取了 `/etc/shadow`（密碼檔案）
- 一個 Pod 開始對外發起奇怪的網路連線
- 有人試圖向 Kubernetes API 發出異常請求

#### Falco 的偵測機制

Falco 透過以下方式收集事件：
1. **eBPF / 核心模組**：即時收集系統呼叫事件
2. **Kubernetes API**：監聽 Kubernetes 的 audit events

規則由三個部分組成：
- **macro**：可重用的條件片段
- **list**：可重用的值列表
- **rule**：實際的偵測規則

#### 規則結構

```yaml
# 規則範例：偵測容器內的 shell 執行
- rule: Terminal Shell in Container
  desc: 容器內有互動式 shell 被開啟
  condition: >
    spawned_process           # 有新程序被建立
    and container             # 在容器內
    and shell_procs           # 是 shell 程序（bash, sh, zsh 等）
    and proc.tty != 0         # 有 TTY（代表是互動式）
  output: >
    Shell opened in container
    (user=%user.name container=%container.name
    image=%container.image.repository
    shell=%proc.name cmdline=%proc.cmdline)
  priority: WARNING
  tags: [container, shell]
```

#### 常用的 Falco Fields

| Field | 說明 |
|-------|------|
| `proc.name` | 程序名稱 |
| `proc.cmdline` | 完整命令列 |
| `proc.pname` | 父程序名稱 |
| `user.name` | 執行程序的使用者 |
| `container.id` | 容器 ID |
| `container.image.repository` | 容器映像名稱 |
| `fd.name` | 被存取的檔案路徑 |
| `evt.type` | 系統呼叫名稱（如 open、execve） |

```bash
# 查看 Falco 告警（安裝在叢集中）
kubectl logs -n falco -l app.kubernetes.io/name=falco -f

# 直接執行 Falco（節點上）
falco -r /etc/falco/falco_rules.yaml

# 查看預設規則
cat /etc/falco/falco_rules.yaml
```

---

### 5.3 偵測不可變性違規

#### 概念：執行期的完整性

即使容器已設定 `readOnlyRootFilesystem: true`，攻擊者仍可能嘗試在可寫入的 mount point（如 `/tmp`）中執行惡意程式。Falco 可以偵測這類行為。

常用 Falco 規則：

```yaml
# 偵測容器內的寫入操作（到不該寫入的目錄）
- rule: Write Below Binary Dir
  desc: 有程序寫入 /bin、/sbin、/usr/bin 等目錄
  condition: >
    bin_dir_write and not package_mgmt_procs and not container_entrypoint
  output: >
    File opened for writing in binary directory
    (user=%user.name command=%proc.cmdline file=%fd.name container=%container.id)
  priority: ERROR

# 偵測從容器內讀取敏感檔案
- rule: Read Sensitive File Untrusted
  desc: 不信任的程序讀取了 /etc/shadow 或 /etc/passwd
  condition: >
    sensitive_files and open_read
    and not proc.name in (trusted_programs)
  output: >
    Sensitive file opened for reading
    (file=%fd.name gparent=%proc.aname[2] program=%proc.name)
  priority: WARNING
```

---

### 5.4 使用 Container Runtime 工具

```bash
# 使用 crictl 查看執行中的容器（適用於 containerd）
crictl ps
crictl inspect <container-id>
crictl logs <container-id>

# 找出某個 Pod 的容器 PID（在節點上）
crictl inspect <container-id> | grep pid

# 進入容器的 namespace 執行命令（調查用）
nsenter --target <pid> --mount --uts --ipc --net --pid -- sh

# 查看容器使用的 system calls（需要 strace）
strace -p <pid> -f -e trace=network 2>&1 | head -50
```

---

## 6. 常用指令速查表

### RBAC 快速建立

```bash
# ServiceAccount
kubectl create serviceaccount my-sa -n default

# Role 和 ClusterRole
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n default
kubectl create clusterrole pod-reader --verb=get,list,watch --resource=pods

# Binding
kubectl create rolebinding my-binding --role=pod-reader --serviceaccount=default:my-sa -n default
kubectl create clusterrolebinding my-binding --clusterrole=pod-reader --serviceaccount=default:my-sa

# 驗證
kubectl auth can-i list pods --as=system:serviceaccount:default:my-sa -n default
kubectl auth can-i --list --as=jane
```

---

### 憑證操作

```bash
# 查看憑證到期時間
kubeadm certs check-expiration
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A 2 Validity

# 建立使用者憑證
openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -subj "/CN=jane/O=developers" -out jane.csr

# 透過 K8s CSR API 簽署
kubectl apply -f - <<EOF
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: $(cat jane.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages: ["client auth"]
EOF

kubectl certificate approve jane
kubectl get csr jane -o jsonpath='{.status.certificate}' | base64 -d > jane.crt
```

---

### AppArmor 快速操作

```bash
# 查看已載入的 profiles
aa-status

# 載入 profile（enforce）
apparmor_parser -q /etc/apparmor.d/my-profile

# 載入 profile（complain 模式，用於測試）
apparmor_parser -C /etc/apparmor.d/my-profile

# 確認 profile 名稱
apparmor_parser -N /etc/apparmor.d/my-profile
```

---

### 重要檔案路徑

| 元件 | 路徑 |
|------|------|
| kube-apiserver | `/etc/kubernetes/manifests/kube-apiserver.yaml` |
| kubelet 設定 | `/var/lib/kubelet/config.yaml` |
| PKI 憑證 | `/etc/kubernetes/pki/` |
| AppArmor profiles | `/etc/apparmor.d/` |
| Seccomp profiles | `/var/lib/kubelet/seccomp/` |
| Falco 規則 | `/etc/falco/falco_rules.yaml` |
| Audit log | `/var/log/kubernetes/audit.log`（自訂路徑） |
| Encryption config | `/etc/kubernetes/enc/encryption-config.yaml`（自訂） |

---

## 附錄：考試前確認清單

### 概念理解確認

- [ ] 能解釋 Authentication → Authorization → Admission Control 的順序和區別
- [ ] 了解 RBAC 四種物件的關係（Role / ClusterRole / RoleBinding / ClusterRoleBinding）
- [ ] 能說明 Network Policy 的預設行為和 podSelector: {} 的含義
- [ ] 了解 AppArmor（profile / enforce / complain）和 Seccomp 的差異與用途
- [ ] 了解 PSS 三種等級和三種模式的差異
- [ ] 了解 etcd 加密的必要性和設定方式
- [ ] 了解 Audit Log 四種 level 的差異
- [ ] 了解 Falco 的工作原理（rule / condition / output）
- [ ] 了解供應鏈攻擊的威脅模型

### 操作能力確認

- [ ] 修改 kube-apiserver 靜態 Pod 設定並等待重啟
- [ ] 建立 NetworkPolicy（default-deny + 精確開放）
- [ ] 建立 RBAC 並以 `auth can-i` 驗證
- [ ] 設定 Pod SecurityContext（多個安全選項組合）
- [ ] 在節點上載入 AppArmor profile 並在 Pod 套用
- [ ] 使用 Trivy 掃描映像並解讀結果
- [ ] 撰寫並套用 Audit Policy
- [ ] 設定 etcd 加密並驗證

---

*最後更新：2024 年 | 適用 Kubernetes v1.29+*
