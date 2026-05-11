# [DAY09] 當一個 Pod 被建立時 Cilium 在背後做了什麼？(3) Identity 的誕生

> 原文連結：https://ithelp.ithome.com.tw/articles/10385922

---

DAY
9

在昨天 (Day 8)，我們深入理解了 Endpoint 如何誕生，可以知道一個 Endpoint 其實裡面的邏輯可說是很複雜的，我認為深入理解 Endpoint 是一個相當重要的課題，畢竟你創建的每一個 Pod 對 CIlium 來說都被抽象成一個個 Endpoint。

接下來我們要深入理解的主角是 — **Identity**，在 「[Day 6] 正式踏入 Cilium 前，先認識這些專有名詞 Cilium 」中，已經知道**每個 Endpoint 都會被分配一個 Identity (是數字)**，那問題來了：Cilium 背後到底怎麼分配這個 Identity？什麼時機決定？Labels 變動時又怎麼處理？

除此之外，Cilium 對於 Network Policy 的實踐正是大大運用了 **Identity** 使其與 iptables-based 的解決方案相比，效能得到大幅提升、且在大規模場景或是 Pod Churn 時都可以怡然自得的應付而不垮台！

今天就把 **Identity 的來源、分配機制、與 Policy 生效的關係**一次講清楚

從 Day 7 開始，我們一路在追問同一件事：**「當一個 Pod 被建立時，Cilium 在背後做了什麼？」**

這條路最明確的終點，就是 **cilium-cni 依照 CNI 規範回覆給 container runtime 一個 Result，Pod 正式具備由 Cilium 完整管理的網路能力**。

那我們到底已經走到哪了呢？來快速複習一下：

-
**Day 7**：我們先看到，當 Pod 被建立時，container runtime 會呼叫 CNI Plugin（這裡是 cilium-cni），這就是整個故事的起點。 -
**Day 8**：在這個過程裡，cilium-cni 會發出一個`PUT /endpoint/{id}`

請求給 cilium-agent，並帶上`SyncBuildEndpoint = true`

，表示這是一個同步阻塞的呼叫，要等到**第一次 Regeneration**完成才能繼續。

所以在 Cilium 的世界裡 ，**Pod 要真正具備網路能力還差哪步？**

**答案是：必須完成它的第一次 Regeneration。**

但是問題來了：第一次 Regeneration 的前提是什麼？

其實我在 Day 8 的 「**Regeneration 是什麼？** 」中，就有提到 Regeneration 的觸發條件有哪些，其中一個就是：

當一個新的 Pod 建立時，它對應的 Endpoint 在成功取得 Identity 之後，就會立刻觸發第一次 Regeneration。


這一步，才是 Pod 從「登記完成」走到「真的能上網」的關鍵。

以下程式碼部分為了避免篇幅太長都有稍作簡化，對完整原始碼感興趣的讀者，我在 Snippet 旁邊都會附上原始碼連結

這裡會有一點點內容是 Day 8 已經講過的部分，畢竟 Cilium Agent 裡面的流程比較複雜，而我又想要告訴讀者

分配 Identity 的起點在哪裡

在 Day 8 的結尾提到：


`AddEndpoint`

這裡是個關鍵分界點，因為它會呼叫`expose()`

，才算是把 Endpoint 正式納入`cilium-agent`

的管理。

接著，對於 Kubernetes Pod，程式會呼叫 `ep.RunMetadataResolver()`


```
// pkg/endpoint/api/endpoint_api_manager.go
err = m.endpointManager.AddEndpoint(ep)
if err != nil {
return m.errorDuringCreation(ep, fmt.Errorf("unable to insert endpoint into manager: %w", err))
}
var regenTriggered bool
if ep.K8sNamespaceAndPodNameIsSet() && m.clientset.IsEnabled() {
// We need to refetch the pod labels again because we have just added
// the endpoint into the endpoint manager...
regenTriggered = ep.RunMetadataResolver(false, true, apiLabels, m.endpointMetadata.FetchK8sMetadataForEndpoint)
} else {
regenTriggered = ep.UpdateLabels(ctx, labels.LabelSourceAny, identityLbls, infoLabels, true)
}
```


關於上方這段程式碼，推薦讀者去看一下原始碼的註解，註解解釋了**如何避免沒有拿到最新 Pod Label 資料的機制**，以下我白話解說該機制：

-
**剛生出一個 Endpoint 的時候**- 這時候它還沒被「註冊」進 Cilium 的管理名單 (endpoint manager)
- 如果這段期間 Kubernetes 有通知「這個 Pod 的 labels 改了」，Cilium 的監聽程式 (event handler) 會想更新，但
**翻名單卻找不到這個 Endpoint**→ 只好放棄

-
**Endpoint 終於被註冊進名單之後**- 為了避免這中間還沒被註冊到 Endpoint Manager 的時間差內有更新 Pod 資料 ( 特別是 Pod Labels )，Cilium 會再主動去「問一次」 K8s，把 Pod 最新的 labels 拿回來，更新到 endpoint
- 所以呼叫
`RunMetadataResolver(false, true, …)`

就是**主動去「問一次」K8s** - 因為這裡設
`blocking=true`

，所以流程會阻塞，直到 Endpoint 確定拿到正確的 labels → 生成 identity → 做完第一次 regeneration，才會繼續往下

- 所以呼叫

- 為了避免這中間還沒被註冊到 Endpoint Manager 的時間差內有更新 Pod 資料 ( 特別是 Pod Labels )，Cilium 會再主動去「問一次」 K8s，把 Pod 最新的 labels 拿回來，更新到 endpoint

我們接著來看看 `RunMetadataResolver`


原始碼連結點我


```
// RunMetadataResolver 會幫 Endpoint 嘗試解析 metadata (labels)，
// 拿到 Identity 之後觸發第一次 Regeneration。
// blocking = true 的話，呼叫端會卡住等結果。
func (e *Endpoint) RunMetadataResolver(restoredEndpoint, blocking bool,
baseLabels labels.Labels, resolveMetadata MetadataResolverCB) (regenTriggered bool) {
var regenTriggeredCh chan bool
callerBlocked := false
if blocking {
regenTriggeredCh = make(chan bool)
callerBlocked = true
}
controllerName := resolveLabels + "-" + e.GetK8sNamespaceAndPodName()
// 啟動一個 controller，執行 metadata 解析邏輯
e.controllers.UpdateController(controllerName,
controller.ControllerParams{
RunInterval: 0, // 僅跑一次
Group: resolveLabelsControllerGroup,
DoFunc: func(ctx context.Context) error {
// 呼叫 metadataResolver → resolve labels → identity → regeneration
regenTriggered, err := e.metadataResolver(
ctx, restoredEndpoint, blocking, baseLabels, resolveMetadata)
if callerBlocked {
select {
case <-e.aliveCtx.Done():
case regenTriggeredCh <- regenTriggered:
close(regenTriggeredCh)
callerBlocked = false
}
}
return err
},
Context: e.aliveCtx,
},
)
// 如果 blocking = true，就等 metadataResolver 第一次跑完再回傳
if blocking {
select {
case regenTriggered, ok := <-regenTriggeredCh:
return regenTriggered && ok
case <-e.aliveCtx.Done():
return false
}
}
return false
}
```


`RunMetadataResolver`

會替這個 Endpoint 啟動一個 controller，去解析 metadata ( 主要是 Pod Labels )，並把它更新回 Endpoint，它會「定期嘗試」，但**一旦成功解析一次，就會停止**。

你一定會想問為什麼成功解析一次就停止？原因如下：

- 回到最初調用
`RunMetadataResolver`

的目的，就是確保剛註冊時 endpoint 不會帶舊 labels，能正確拿最新的 labels，然後確保跑完第一次 regeneration - 避免兩套機制重疊，後續持續變更交給 watcher ****就好

到這邊為止，概觀的流程大概是這樣：

```
CreateEndpoint -> RunMetadataResolver -> (背景 controller) -> metadataResolver -> UpdateLabels
```


我們回到 `pkg/endpoint/api/endpoint_api_manager.go`

繼續往下看程式碼，接著會遇到「**同步等待」**：在觸發完 Label 更新和 Regeneration 流程後，`CreateEndpoint`

函式會檢查 CNI 傳來的 `SyncBuildEndpoint`

Flag。因為 cilium-cni 設定了此 Flag 為 `true`

，cilium-agent 會在這裡執行一個**阻塞操作**

原始碼連結點我


```
// pkg/endpoint/api/endpoint_api_manager.go
// ... (前面的步驟) ...
if epTemplate.SyncBuildEndpoint {
if err := ep.WaitForFirstRegeneration(ctx); err != nil {
return m.errorDuringCreation(ep, err)
}
}
return ep, 0, nil
```


`ep.WaitForFirstRegeneration()`

函式會徹底阻塞 `CreateEndpoint`

的執行緒，直到這個新 Endpoint 的 BPF 程式被成功編譯、加載並應用到 datapath（即 `policyRevision > 0`

）。只有在 Identity 分配、BPF Regeneration、datapath 配置**全部完成**後，這個函式才會返回，`CreateEndpoint`

API 呼叫也才會向 CNI Plugin 返回成功

Identity 的分配是在 `pkg/endpoint/endpoint.go`

中完成的。`UpdateLabels`

方法是這一切的起點：

原始碼連結點我


`UpdateLabels`

的設計定位：

- 在 Cilium 裡，
**Endpoint 的 identity 完全取決於 labels** - 不管是
**第一次建立 Endpoint**（一開始沒有 labels，要設定第一組），還是**後續 Pod labels 改變**（舊的換成新的）

→ 本質上都是「**把最新的 labels 狀態，同步到 Endpoint 上**」

```
// pkg/endpoint/endpoint.go
func (e *Endpoint) UpdateLabels(ctx context.Context, sourceFilter string, identityLabels, infoLabels labels.Labels, blocking bool) (regenTriggered bool) {
// ... log 記錄 ... 略 ...
if err := e.lockAlive(); err != nil {
// ... 略 ...
return false
}
e.replaceInformationLabels(sourceFilter, infoLabels)
// 替換 Identity 相關 Label，如果發生變更，rev 將不為 0
rev := e.replaceIdentityLabels(sourceFilter, identityLabels)
// ... 處理 reserved:init Label ...
e.unlock()
if rev != 0 {
// Label 已變更，啟動 Identity 解析流程
return e.runIdentityResolver(ctx, blocking, 0)
}
return false
}
```


`UpdateLabels`

的核心職責

-
**接收新的 labels** -
**跟舊的比對**→ 看有沒有差異 - 如果 labels 有改變：
-
**重新計算 identity** -
**觸發 regeneration**（因為 identity 變了，BPF policy 要重編）

-

所以上面 `UpdateLabels`

的程式碼核心在於 `replaceIdentityLabels`

，它會比較新舊 Labels，若有不同，則增加 `identityRevision`


`UpdateLabels`

檢測到 `identityRevision`

變化後，便會呼叫 `runIdentityResolver`

：

-
`runIdentityResolver`

會啟動一個背景 controller，其核心任務是執行`identityLabelsChanged`

函式 -
`identityLabelsChanged`

負責實際的 Identity 分配原始碼連結點我

`// pkg/endpoint/endpoint.go func (e *Endpoint) identityLabelsChanged(ctx context.Context) (regenTriggered bool, err error) { // ... lock 與前置檢查 ... // 如果 Label 與當前 Identity 的 Label 相同，則無需分配 if e.SecurityIdentity != nil && e.SecurityIdentity.Labels.Equals(newLabels) { // ... return false, nil } // ... 中間略 ... // 呼叫 allocator 進行分配 allocatedIdentity, _, err := e.allocator.AllocateIdentity(ctx, newLabels, true, identity.InvalidIdentity) if err != nil { // ... (錯誤處理) ... return false, err } // 將新分配的 Identity 設定到 Endpoint 上 e.SetIdentity(allocatedIdentity, false) // ... Release 舊 Identity ... // 觸發 Endpoint 的 Regeneration if e.ID != 0 { readyToRegenerate = e.setRegenerateStateLocked(regenMetadata) } if readyToRegenerate { e.Regenerate(regenMetadata) } return readyToRegenerate, nil }`


最關鍵的一行就是 `e.allocator.AllocateIdentity()`


這裡的 `allocator`

是 `cache.IdentityAllocator`

的 instance，它負責跟 backend storage（KVStore 或 CRD）打交道。整個流程可以拆成幾個步驟：

-
**查詢 / 分配 Identity**`AllocateIdentity`

會拿著 Endpoint 的 labels 去查後端。- 如果已經有對應的 Identity，就直接回傳
- 如果沒有，就會 atomic 地宣告一個新的、全 cluster 唯一的 Numeric Identity，並把對應關係存起來

-
**更新 SelectorCache（關鍵副作用）**SelectorCache 這部分聽不太懂沒關係，之後我們的的文章會講到 Network Policy，就會講解 SelectorCache 是什麼

- 在
`identityLabelsChanged`

這個 function 裡，呼叫`AllocateIdentity`

時會帶`notifySelectorCache = true`

- 這代表它在拿到 Identity 的同時，會同步更新本地的
`SelectorCache`

-
`SelectorCache`

是 Cilium policy engine 的核心元件，維護著「Label Selector → Identity 列表」的映射。 - 有了這個即時更新，Cilium 的 Network Policy 才能立刻感知新 Pod 的存在，並把它納入 policy 計算

- 在
-
**更新 Endpoint**- 一旦拿到
`allocatedIdentity`

，會直接設定到 Endpoint object 上

- 一旦拿到
-
**觸發 Regeneration**- 最後呼叫
`e.Regenerate()`

，這會重新生成 Endpoint 的 BPF 程式與 Policy Map。新的 Security Identity 就會被寫進 BPF map，真正生效在 datapath 層

- 最後呼叫

Cilium 在幫新 Pod 分配 Identity 的流程，其實是一個橫跨 **CNI plugin** 跟 **cilium-agent** 的 **同步阻塞機制**，設計得非常精細。

-
**CNI plugin**先負責把網路基礎設施建好，然後對 cilium-agent 發送`EndpointCreate`

請求，並帶上`SyncBuildEndpoint`

這個同步 flag（超重要，這幾天的文章我反覆提過） -
**cilium-agent**的 API Server 收到請求後，交給`endpointAPIManager`

處理 -
**endpointAPIManager**會建立一個 Endpoint object，並透過`RunMetadataResolver`

啟動非同步的 label 解析，最後呼叫到`UpdateLabels`

-
發現 labels 有變，就會觸發`UpdateLabels`

`runIdentityResolver`

-
會呼叫`identityLabelsChanged`

`AllocateIdentity`

，到後端存放區去拿或建立一個 cluster 唯一的 numeric identity，並立刻更新本地的 SelectorCache - Endpoint 一旦拿到新的 identity，就會觸發
**Regenerate**。而這裡的關鍵是：因為 CNI 請求裡設了`SyncBuildEndpoint`

，cilium-agent 的`EndpointCreate`

流程會整個**同步阻塞等待**，直到這次 Regeneration**完全成功**。只有 datapath 確實生效後，cilium-agent 才會回覆 CNI plugin 成功，最後完成 Pod 的網路建立

這個同步設計保證了：**當 kubelet 說 Pod 網路 ready 的時候，對應的 NetworkPolicy 已經在 datapath 層生效**。換句話說，Pod 一出生就馬上受到 Policy 保護，不會有「啟動初期還沒套上規則」的空窗期。

