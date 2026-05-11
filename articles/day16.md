# [DAY16] Cilium Network Policy 與 Identity-based 入門

> 原文連結：https://ithelp.ithome.com.tw/articles/10391011

---

DAY
16

在 Day 12 ~ Day 15 我們深入探索了 Cilium 的 Pod-to-Pod datapath，從 Day 14 的網路介面架構到 Day 15 的跨 Node 流量分析，我們看到了 eBPF 如何在核心空間實現高效的封包處理。但是，**網路連通性只是故事的一半**，在企業中，**安全性**才是真正的挑戰。

現在請試想一個情境：由於安全因素 server Pod 裡面的資源比較機敏，你不希望 server Pod 被其他 Pod 存取，只有 Client Pod 可以向 Server Pod 發出 HTTP 請求，那你會怎麼做？

我第一個想法會很直覺地使用 iptables，寫個 rule 僅允許 client Pod 的 IP 的流量允許打到 server Pod 就好，這樣做完全沒問題，但是在 K8s 的場景卻會有一大難題 — 「Pod 變化速度快，IP 隨時會有變動」。

試想我現在有 10 個 server Pods 就好，如果今天我 Deploy 新的版本，那就會發生創建 10 新的個 Pod + 刪除 10 個舊的 Pod，背後換來的是大量 iptables rules 的更新，更別想真實戰場上隨便都超過 10 個 Pods 。

現在回頭看到我們的需求：「只有 Client Pod 可以向 Server Pod 發出 HTTP 請求」，說真的這些 Pod 背後唯一的 IP 真的那麼重要嗎？真實世界中如果校規規定「只有資工系學生可以進入學校機房」你會怎麼做？100 個學生現在走到機房門口一個一個向你出示學生證，你會看得是「學號」還是「系所」來決定眼前這位學生是否可以進入機房？在你的大腦中如果你是用學號去決定那就可累了… 因為每年畢業一屆然後招一屆新生你大腦就要重新更新一次哪些學號可以進去機房，如果讓事情變簡單一點，你只要記著「資工系」「可以進去」「機房」，那即便每年畢業和招新生對你來說都不會有影響，因為畢業的人會被學校註銷在學資工系的身份，新生會被賦予資工系的身份。

我希望這一個現實生活的比喻可以點出 Cilium Identity-based 的概念

在 **[Day 6] 正式踏入 Cilium 前，先認識這些專有名詞**，其實我們已經有講到：

Identity = 一組 Security Relevant Labels → 換句話說，有相同 Security Relevant Labels 的 Endpoints，

會拿到同一個 Identity

也在 Day 6 提及「什麼是 **Security Relevant Labels?」**

所以建議讀者可以回到 Day 6 的文章稍作複習，這裡我將展示 Pod 的實際案例，以及計算方式

-
**相同 Labels 組合 = 相同 Identity**`# 這兩個 Pod 會有相同的 Identity Pod A: {app: web, env: prod} → Identity: 26328 Pod B: {app: web, env: prod} → Identity: 26328 # Labels 組合不同就是不同的 Identity Pod C: {app: api, env: prod} → Identity: 45627`

-
**Identity 數字是如何產生的？**- Cilium 會將 Labels 排序後組合成一個「指紋 (fingerprint) 」
- 相同的 Labels 組合永遠產生相同的指紋
- 指紋對應到一個唯一的數字 ID

-
**為什麼要用數字而不直接用 Labels？**-
**效能考量**：在 BPF 程式中比較數字比比較字串快很多 -
**空間節省**：一個 32-bit 數字比一堆 Label 字串小很多 -
**快速查詢**：Policy Map 可以用數字做高效的鍵值查詢

-

再次回到需求：只有 Client Pod 可以向 Server Pod 發出 HTTP 請求

我們將使用 NetworkPolicy 來滿足需求！

初入 Cilium 想要使用 Network Policy 時，一定會遇到這些問題：

- NetworkPolicy, CiliumNetworkPolicy (CNP) 和 CiliumClusterwideNetworkPolicy (CCNP) 的差異是什麼？
- 那這三個我該用哪個？

以下我整理了一個表格：

| 特性 | NetworkPolicy |
CiliumNetworkPolicy (CNP) |
CiliumClusterwideNetworkPolicy (CCNP) |
|---|---|---|---|
資源類型 |
Kubernetes 原生資源 | Cilium CRD | Cilium CRD |
範圍 (Scope) |
Namespaced（僅限某個 Namespace） | Namespaced | Cluster-wide（跨所有 Namespace） |
支援層級 |
L3 / L4（IP, Port） | L3 ~ L7（HTTP, gRPC, Kafka, DNS, FQDN 等） | L3 ~ L7（同 CNP） |
Ingress / Egress |
✅ 支援 | ✅ 支援 | ✅ 支援 |
FQDN 規則 |
❌ 不支援 | ✅ 支援（例如 `github.com` ） |
✅ 支援 |
L7 控制 |
❌ 不支援 | ✅ 支援（HTTP Method, Path, Header 等） | ✅ 支援 |
跨 Namespace 控管 |
❌ 每個 Namespace 都要寫一份 | ❌ 侷限單一 Namespace | ✅ 一次規則全 Cluster 生效 |
NodeSelector 支援 |
❌ | ❌ | ✅ 可以針對 Node 流量設規則 |
常見使用情境 |
基礎 L3/L4 防護 | 細粒度的 L7 控制、FQDN 安全 Policy | 全域性的安全規範、基礎防護（跨 Namespace） |

從上面表格的觀察來看，如果我們需求是要針對 HTTP 請求，這是屬於 **L7** 的範疇，但原生 **NetworkPolicy 不支援 L7**

👉 所以我要從 CNP 和 CCNP 來選擇

為什麼 Cilium 可以支援 L7？我會在後面的篇章揭曉！


而我只想控制特定 Namespace 的 Pod，所以我最終選擇「**CNP**」

接著有一個地方要特別注意，假如某 Endpoint 身上還沒有被套任何 Policy，那此 Endpoint 向其他 Endpoint 發出流量，此流量到底會被 Allow 還是 Deny?

這題的答案就是看 **Policy Enforcement Modes**，這可以用來決定「**整個 cluster 裡的 endpoints 在沒有明確 policy 時，要不要強制啟用防護**」

有三種模式：

-
**default (預設)**- 沒有 policy → endpoint 全開放（allow all）
- 一旦某個 endpoint 被 policy 選到 → 該 endpoint 的方向（ingress/egress）會進入
**default-deny**，只允許 policy 明確允許的流量 -
**效果**：安全但不會一上來就全封鎖，只有被 policy 選到的 endpoint 會進入「鎖起來」模式

-
**always**- 無論有沒有 policy，所有 endpoint 一律進入
**default-deny** - 換句話說，
**一旦開啟 always，你必須寫明確的 allow rules，不然所有 Pod 流量都會被擋掉** - 適合高安全環境（零信任，先全擋，再逐步放行）

- 無論有沒有 policy，所有 endpoint 一律進入
-
**never**- 完全不 enforce policy，所有流量都 allow
- 基本等於「Cilium 當普通 CNI 用，不管 security」


我們以 **default 模式** 為例，Cilium 是「按需」進入 default-deny：

- Pod 啟動 → 沒有任何 policy select → allow all
- 第一個 policy select 到這個 Pod，如果裡面有
**ingress**section → Pod 的 ingress 變成 default-deny（不在 policy 允許的全擋） - 如果 policy 有
**egress**section → Pod 的 egress 也變成 default-deny

👉 換句話說：Cilium 在 default 模式下，是「lazy activation」的 default-deny

我們可以用以下指令找到現在使用什麼 Mode:

```
$ kubectl get cm -n kube-system cilium-config -o yaml | grep enable-policy
enable-policy: default
```


現在我的 K8s Cluster 有三個 Pod，如下：

```
$ kubectl get po --show-labels
NAME READY STATUS RESTARTS AGE LABELS
client 1/1 Running 119 (6m54s ago) 5d1h app=client
hacker 1/1 Running 0 43s app=hacker
server 1/1 Running 0 26h app=server
```


我們的需求是：僅允許 client Pod 向 server Pod 發出 HTTP 請求：

```
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
name: "allow-client-http-policy"
spec:
description: "Allow HTTP Request from app=client"
endpointSelector: # Policy 套在哪個 Endpoint 身上
matchLabels: # 套在符合以下 Labels 的 Endpoint
app: server
ingress:
- fromEndpoints:
- matchLabels:
app: client # 允許 client Endpoint 的 Ingress 流量
toPorts:
- ports:
- port: "80" # HTTP 80 port
protocol: TCP
rules:
http: # 配置 HTTP 相關 Policy
- method: "GET"
path: "/"
```


Apply 上方的 CNP 後，我們可以使用 client Pod 來測試看看：

```
# 使用 client Pod 向 server Pod 發送 HTTP Request
$ kubectl exec client -- curl -s 10.244.1.9
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```


```
# 使用 hacker Pod 向 server Pod 發出 HTTP Request
$ kubectl exec hacker -- curl -s 10.244.1.9 --max-time 10
command terminated with exit code 28
```


結果正如上面，我們成功滿足需求：

- ✅
**僅允許 client Pod 向 server Pod 發出 HTTP 請求**

另外，我想再多補充一點範例，CNP 還可以允許特定 Header，如下

```
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
name: "allow-client-http-policy"
spec:
description: "Allow HTTP Request from app=client"
endpointSelector:
matchLabels:
app: server
ingress:
- fromEndpoints:
- matchLabels:
app: client
toPorts:
- ports:
- port: "80"
protocol: TCP
rules:
http:
- method: "GET"
path: "/"
headers:
- 'X-Secret-Header: secret-value' # 添加了這行
```


套用了該 Policy 後，可以在用 client pod 發出請求試試看：

```
# 使用 client Pod 向 server Pod 發送 HTTP Request
$ kubectl exec client -- curl -s 10.244.1.9
Access denied
```


會發現直接顯示 **Access denied**，並不像 hacker Pod 直接卡在那邊直到 Timeout，這是就是 L3/4 和 L7 不同的行為

| Pod | Label 匹配 | Policy 執行層級 | 結果 |
|---|---|---|---|
`client` |
✅ 符合 `fromEndpoints` Selector |
L7 HTTP Proxy |
立即回應 "Access denied" |
`hacker` |
❌ 不符合 Policy Selector | L3/L4 BPF Drop |
封包靜默丟棄 → 連線 Timeout |

而我們發出請求攜帶指定 Header 就可以成功被放行流量：

```
# 使用 client Pod 向 server Pod 發送 HTTP Request，請求攜帶特定 Header
$ kubectl exec client -- curl -s -H "X-Secret-Header: secret-value" 10.244.1.9
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```


回顧我們之前在 Day 12 ~ Day 15 探討的 Datapath，其實我們在看原始碼過程中就已經看到很多地方有做「Policy 檢查」

像是我們討論過的 `handle_ipv4_from_lxc()`

，仔細看會發現 **Policy 檢查發生在路由決策之前**：

原始碼連結點我


```
// bpf/bpf_lxc.c - handle_ipv4_from_lxc()
__u8 policy_match_type = POLICY_MATCH_NONE;
__u8 audited = 0;
__u8 auth_type = 0;
__u16 proxy_port = 0;
int verdict;
verdict = policy_can_egress4(
ctx, &cilium_policy_v2, tuple, l4_off, SECLABEL_IPV4,
*dst_sec_identity, &policy_match_type, &audited,
ext_err, &proxy_port);
if (verdict == DROP_POLICY_AUTH_REQUIRED) {
__u32 tunnel_endpoint = 0;
auth_type = (__u8)*ext_err;
if (info)
tunnel_endpoint = info->tunnel_endpoint.ip4;
verdict = auth_lookup(
ctx, SECLABEL_IPV4, *dst_sec_identity,
tunnel_endpoint, auth_type);
}
// 發送 Policy 決策通知
if (verdict != CTX_ACT_OK || ct_status != CT_ESTABLISHED) {
send_policy_verdict_notify(
ctx, *dst_sec_identity, tuple->dport,
tuple->nexthdr, POLICY_EGRESS, 0, verdict,
proxy_port, policy_match_type, audited, auth_type);
}
if (verdict != CTX_ACT_OK)
return verdict;
// 如果需要 L7 proxy 處理
if (proxy_port > 0) {
return ctx_redirect_to_proxy4(ctx, tuple, proxy_port, false);
}
// Policy 允許後才執行後續處理邏輯
```


這段落主要是想讓讀者回顧「在之前探索 Datapath 時，我們就有看到 Policy 怎麼影響著封包的流動」，至於更多 CIlium Network Policy 底層實作細節，就不在今天這篇展開了

這幾天我們一路從 Pod-to-Pod datapath 拆到跨 Node 的流量，看到 eBPF 怎麼讓封包在核心裡被攔下、轉送。但光「流得通」還不夠，**安全性**才是真正的挑戰。

在 K8s 世界裡，Pod IP 變動太快，靠 IP 寫規則完全不實際。Cilium 把這件事抽象成 **Identity**：不是看 IP，而是看 Labels。這就像進機房不是看學號，而是看「是不是資工系學生」。只要規則基於身份，就不用管底層 IP 怎麼變。

所以 Cilium 不只是把封包送到對的地方，更是確保「對的人能做對的事」，這就是 Identity-based Security 的核心價值

系列文

30 天深入淺出 Cilium ：從入門到實戰
共 30 篇

-
26[Day 26] Cilium Cluster Mesh 實現跨 Cluster 高可用與安全性： Load Balancing 與 NetworkPolicy 實踐
-
27[Day 27] Cilium 實戰分享 (1) 裝了 NodeLocalDNS Cache， DNS 封包原來都沒進去 NodeLocalDNS Cache？
-
28[Day 28] Cilium 實戰分享 (2) 想監控 DNS，封包確實送進去 NodeLocalDNS Cache 了， 但是 hubble_dns_queries_total 怎沒計算到？
-
29[Day 29] Cilium 實戰分享 (3) 裝了 Cilium 後，流量來了，一個 Pod 要 Ready 需要等 26 分鐘？
-
30[Day 30] 深入淺出 Cilium 完結篇：從 Cilium 出發的下一段旅程
