# [DAY06] 正式踏入 Cilium 前，先認識這些專有名詞

> 原文連結：https://ithelp.ithome.com.tw/articles/10383818

---

DAY
6

在 Day4, Day 5 我們已經認識 Cilium 底層所依賴的核心技術 — eBPF，在我們正式踏入 Cilium 前，我們也必須認識一些 Cilium 才有的專有名詞，這樣在後面的文章講解 Datapath、運作原理等 …，我才不用中途塞一個篇幅去說明這個 XXX (專有名詞) 是什麼

這篇文章就像是一個小字典，如果後續有新的專有名詞我也會同步更新上來

在 K8s 的上下文中，我們常說 Pod，但是在 Cilium 的上下文中，你會很常聽到 Endpoint

簡單來說，**Endpoint = Pod 在 Cilium 的抽象實體**

- 一個 Pod（可能包含多個 Container）會對應到一個 Endpoint
- Endpoint 會被分配到 IP
- Endpoint 有獨立的
**Endpoint ID**與**Metadata**，Cilium 就靠這些來做流量控制

每個 Pod 在被管理時，Cilium 都會幫它分配一個 Endpoint ID，並且在 eBPF Map 裡記錄它的資訊

假設我們建立 `client`

Pod 之後可以透過以下指令找到 `CiliumEndpoint`

:

```
$ kubectl get ciliumendpoint client
NAME SECURITY IDENTITY ENDPOINT STATE IPV4 IPV6
client 26328 ready 10.244.2.16
server 9764 ready 10.244.2.135
```


Node 就是 Kubernetes Cluster 裡的一台機器（Worker / Control-plane 都算）

每個 Node 都會有一個 `CiliumNode`


cilium-agent 是 DaemonSet ，所以每個 Node 身上都會有 cilium-agent，cilium-agent 會為他所處的 Node 建立 `CiliumNode`


可以透過以下指令找到 `CiliumNode`

:

```
$ kubectl get ciliumnode
NAME CILIUMINTERNALIP INTERNALIP AGE
control-plane 10.244.0.36 10.13.1.243 4d19h
worker-1 10.244.2.161 10.13.1.57 4d18h
worker-2 10.244.1.51 10.13.1.129 4d19h
```


我相信你看到這裡會覺得這是什麼專有名詞？不就是 K8s 裡的 `metadata.labels`

，對，沒理解錯，但只可以說對一半

在 Cilium：Label 是 **安全性與網路的基礎單位**，因為 Cilium **Identity** 就是由一組 Labels 計算出來的

嚴格來說是一組

Security Relevant Labels 計算出一個 Identity，這裡先簡化解說

而 Cilium 取得 Label 的來源包括：

-
`k8s`

→ Kubernetes`metadata.labels`

-
`container`

→ Container runtime label (e.g. Docker run 加`l app=foo`

) -
`reserved`

→ Cilium 保留的特殊標籤 (world, host, kube-apiserver…) -
`unspec`

→ 其他不明確來源

所以 Cilium Label 其實是「抽象化的超集合 (superset)」，能包進去多種來源，而不是單純的 k8s Label

每個 Endpoint 都會被分配一個 **Identity (是數字)**

Identity = 一組 Security Relevant Labels → 換句話說，有相同 Security Relevant Labels 的 Endpoints，**會拿到同一個 Identity**

觀察 `get ciliumendpoint`

的 Output，可以注意到有一個 `SECURITY IDENTITY`

這就是該 Endpoint 被分配到的 Identity:

```
$ kubectl get ciliumendpoint client
NAME SECURITY IDENTITY ENDPOINT STATE IPV4 IPV6
client 26328 ready 10.244.2.16
server 9764 ready 10.244.2.135
```


有一些東西可能沒有 Labels 可以拿，那就沒辦法計算出 Identity，這種情況就衍生出一個 Identity 叫做 **Special Identities** 會以 `reserved:`

開頭，詳細可以看這份文件，我底下簡單舉例一下：

-
假設你想要

**阻擋**某 Endpoint 訪問 K8s Cluster 外的東西 -
那在 Cilium Network Policy 可以這樣寫:

`toEntities: - world # reserved:world`

-
注意到上面了嗎，這其實不是 match Pod labels (根本沒 Label 計算 Identity…)，而是直接用

**Special Identitiy**`reserved:world`

來代表「Cluster 外的東西」

看完上面的 Identity，我猜你接下來會想問，什麼是 **Security Relevant Labels?**

你可以想像一下，如果你的某個 application 有一個 Label 叫做 `deploy-timestamp`

，你每次部署新版本，就會導致 Identity 被變換，但明明這跟 Security 無關，後果就是讓你同一組應用各自都拿到不同的 Identity，也會導致 Identity 暴長一堆。

如果 **所有 labels 都算進 Identity**，會出大問題：

- Pod Deploy 時間戳不同 → Identity 就變了 → Identity 老是因為「不重要的 Label」而變化，Policy 要一直重新計算、下放到 BPF map →
**浪費效能、Map 空間** - 實際上我們只在乎「service 是 frontend」這種 label，不在乎「什麼時間 deploy 」

所以 Cilium 允許你設定一個 **prefix 白名單**，例如 `id.`

，只有這些 prefix 開頭的 Label，才會被算進 Identity 推導，這樣就不會因為 Pod 上隨便加一些 metadata（像 `timestamp`

）而搞出一堆不必要的 Identity

`CiliumNetworkPolicy`

(CNP) 是為了補 K8s NetworkPolicy 不夠用而存在的「加強版」

Cilium 實現的 Policy 本質上就是 **用 Label+Identity 選擇哪些 Endpoint 可以通，所以你才會聽到 Identity-based Policy**

CNP 比 K8s 原生 NetworkPolicy 更強是強在哪裡呢？

- K8s 原生的 NetworkPolicy
- 只能做到
**L3/L4**（IP + Port），不支援**L7** - 無法指定
**DNS-based rule**（例如允許`api.shiun.me`

)

- 只能做到
- 所以 Cilium 才自己定了 CRD，來彌補原生的 NetworkPolicy 不足之處

Cilium 的世界有兩個 CLI Tools ( `cilium`

, `cilium-dbg`

) 很常用，兩個很常搞混，若沒提前解釋這兩者的差異，怕之後的文章會讓讀者混淆，甚至不知道什麼場合該使用哪個 CLI

不是跑在 Pod 裡，而**是你在本機 (laptop 或管理機器) 安裝的 Binary**，安裝方式可以參考文件。

**主要用途**：

- 用來安裝 / upgrade / uninstall Cilium (
`cilium install`

,`cilium status`

) - 做 cluster-wide 的 debug (
`cilium connectivity test`

) - 管理 Hubble (
`cilium hubble enable`

)

是在每個 Node 上的 `cilium-agent`

Pod 裡面的附帶的 CLI，**你要進到 Cilium Pod 或 node 上才用得到。**

**主要用途**：

- 跟
**本機 Node 上的 Cilium agent**溝通 - 可以看
**BPF maps**、**Endpoint 狀態**、**Policy 生效**

關於 Cilium CNI, Cilium Agent, Cilium Operator 敬請參考 Day 3 的文章！

