# [DAY08] 當一個 Pod 被建立時 Cilium 在背後做了什麼？(2) Cilium Agent 的職責與 Endpoint 的誕生

> 原文連結：https://ithelp.ithome.com.tw/articles/10385123

---

DAY
8

Cloud Native
### 30 天深入淺出 Cilium ：從入門到實戰系列 第
8 篇

##
[Day 8] 當一個 Pod 被建立時 Cilium 在背後做了什麼？(2) Cilium Agent 的職責與 Endpoint 的誕生


在昨天 (Day 7) 我們已經知道了 container runtime 會呼叫 cilium-cni，而這是 Cilium 工作流程的起點，同時也知道了 cilium-cni 只負責了很基本的網路工程 (例如：建立 veth pair, 移動網卡到 Pod Network namespace 等)，許多重要的任務都調用了 API 委派給 cilium-agent。

在沒有深入認識 Cilium 的運作之前，其實我以為 cilium-cni 會負責很多任務，結果實際上並非如此。

今天的文章，核心目標是想要來帶大家**認識 Endpoint 的誕生，**還記得嗎？Endpoint 的誕生的起點來自於 cilium-cni 向 cilium-agent 發出 `PUT /endpoint/{id}`

請求，就如下圖所示：

那麼，這個 API 請求進入 `cilium-agent`

後，到底發生了什麼事？這篇文章，我們就來深入解析 **Cilium Agent 的內部流程**，以及 Endpoint 在這裡是如何誕生的。

因為這裡比較複雜且有 Cilium 的 know-how，不像是 CNI 是已經有一個標準規範很容易預測 CNI 接下來可能還會做哪些操作，也因此在開頭我先把完整流程圖先呈現出來，接著再慢慢拆解每個細節

待會會看到一個名詞叫做 “**Regeneration**”，認識這個名詞的意義對今天這篇文章來說非常重要！

在 Cilium 的世界裡，**Regeneration 就是將期望的網路狀態同步到實際的 BPF Datapath 的過程**

而 Regeneration 並非隨時發生，而是由**特定事件觸發，這些事件通常意味著 Endpoint 的期望狀態發生了改變**，觸發點包括：

-
**Endpoint 首次建立**：當一個新的 Pod 被建立，其對應的 Endpoint 在成功獲取 Identity 後，會立即觸發第一次 Regeneration，這是將其納入網路管理的關鍵一步 -
**Identity 變更**：當 Pod 的 Labels 發生變化，可能導致其 Identity 改變。因為 Cilium 的 Policy 是基於 Identity 的，所以 Identity 變更必須觸發 Regeneration 來重新計算並應用新的 Network Policy。這個邏輯主要在`pkg/endpoint/endpoint.go`

的`runIdentityResolver()`

中處理 -
**Network Policy 更新**：建立、刪除或修改`CiliumNetworkPolicy`

或`CiliumClusterwideNetworkPolicy`

時，cilium-agent 會識別所有受影響的 Endpoints，並為它們觸發 Regeneration -
**Endpoint 設定變更**：透過 API 或`cilium`

CLI 修改 Endpoint 的特定設定（例如，啟用/停用某個功能）時，也會觸發 Regeneration 以使新設定生效

*圖片取自：https://docs.cilium.io/en/stable/_images/cilium-endpoint-lifecycle.png*

當 Endpoint 被建立並開始被管理後，它會經歷一個完整的生命週期，**每個 Cilium Endpoint 都會處於以下其中一種狀態**：

-
**restoring**：這個 Endpoint 在 Cilium 啟動之前就已經存在，Cilium 正在恢復它的網路配置 -
**waiting-for-identity**：Cilium 正在為這個 Endpoint 分配唯一的 identity -
**waiting-to-regenerate**：Endpoint 已經拿到 identity，正在等待它的網路配置被 regenerate -
**regenerating**：Endpoint 的網路配置正在被生成，這包含 eBPF 的程式化 -
**ready**：Endpoint 的網路配置已經成功完成生成，處於可用狀態 -
**disconnecting**：Endpoint 正在被刪除 -
**disconnected**：Endpoint 已經被刪除

我們可以透過

`cilium-dbg endpoint list`

和`cilium-dbg endpoint get`

CLI 指令來查詢 Endpoint 的狀態

實際的原始碼比較複雜，而且檔案會跳來跳去的，所以這裡針對原始碼解析有稍作簡化

`cilium-agent`

內建了一個 **基於 go-swagger 的 API 伺服器**，它會把 `PUT /endpoint/{id}`

路由綁定到 `PutEndpointIDHandler`

，再交由 handler 處理。

```
// api/v1/server/server.go
func newAPI(p apiParams) *restapi.CiliumAPIAPI {
api := restapi.NewCiliumAPIAPI(p.Spec.Document)
// ...
// 將 PUT /endpoint/{id} path 綁定到 PutEndpointIDHandler
api.EndpointPutEndpointIDHandler = p.EndpointPutEndpointIDHandler
// ...
return api
}
```


原始碼連結點我


真正的處理邏輯在 `pkg/endpoint/api/endpoint_api_handler.go`


```
// pkg/endpoint/api/endpoint_api_handler.go
type EndpointPutEndpointIDHandler struct {
logger *slog.Logger
apiLimiterSet *rate.APILimiterSet
endpointAPIManager EndpointAPIManager // <--- 核心依賴
}
func (h *EndpointPutEndpointIDHandler) Handle(params endpointapi.PutEndpointIDParams) (resp middleware.Responder) {
epTemplate := params.Endpoint
// ... 省略細節... 速率限制等前置作業 ...
// 將請求轉發給 endpointAPIManager 進行實際處理
ep, code, err := h.endpointAPIManager.CreateEndpoint(params.HTTPRequest.Context(), epTemplate)
if err != nil {
return api.Error(code, err)
}
ep.Logger(endpointAPIModuleID).Info("Successful endpoint creation")
return endpointapi.NewPutEndpointIDCreated().WithPayload(ep.GetModel())
}
```


Handler 幾乎不做事，重點在**統一入口、驗證、速率限制**，然後**轉交** Manager

`endpointAPIManager`

統籌建立過程原始碼連結點我


這裡的 `CreateEndpoint`

函式是整個建立流程的核心協調者，而且這 Function 的行數也比較長！

```
// pkg/endpoint/api/endpoint_api_manager.go
type endpointAPIManager struct {
// ...
endpointManager endpointmanager.EndpointManager
endpointCreator endpointcreator.EndpointCreator
// ...
}
func (m *endpointAPIManager) CreateEndpoint(ctx context.Context, epTemplate *models.EndpointChangeRequest) (*endpoint.Endpoint, int, error) {
// 1. 透過 EndpointCreator 從請求模型建立一個初步的 Endpoint 物件
ep, err := m.endpointCreator.NewEndpointFromChangeModel(ctx, epTemplate)
if err != nil {
// ... 錯誤處理 ...
}
// 2. 檢查 ID 和 IP 是否已存在，防止重複建立
oldEp := m.endpointManager.LookupCiliumID(ep.ID)
if oldEp != nil {
// ... 錯誤處理 ...
}
// ... 其他檢查 ...
// 3. 獲取 Kubernetes labels 並合併
// ...
// 4. 將 Endpoint 加入 EndpointManager 進行管理
err = m.endpointManager.AddEndpoint(ep)
if err != nil {
return m.errorDuringCreation(ep, fmt.Errorf("unable to insert endpoint into manager: %w", err))
}
// 5. 觸發第一次的 Regeneration
// 這會開始解析 Identity、計算 Policy 並最終編譯 BPF 程式
// ...
if epTemplate.SyncBuildEndpoint {
if err := ep.WaitForFirstRegeneration(ctx); err != nil {
return m.errorDuringCreation(ep, err)
}
}
return ep, 0, nil
}
```


我們來看看 `CreateEndpoint`

，這邊其實做的事情很明確，但步驟拆開來看會比較清楚。

一開始它不會自己 new 一個 Endpoint，而是把這件事丟給 `EndpointCreator`

。接著馬上會做一些 check，像是看這個 ID 或 IP 有沒有跟現有的 Endpoint 衝突，如果撞到了就直接中斷。

檢查都過了之後，它會把這個 Endpoint 交給 `EndpointManager`

，讓它進入後續的管理流程。

最後還有一個關鍵細節：因為 CNI 那邊在送 request 的時候有帶 `SyncBuildEndpoint=true`

，忘記 `SyncBuildEndpoint`

是什麼的可以回去 「Day 7 - 2.7 呼叫 EndpointCreate」 複習一下，所以這裡不會馬上回成功，而是會去等第一次 Regeneration 跑完，包含 identity 配發、policy 計算、BPF 編譯跟載入，至少要把這些基礎的 datapath 東西弄好才會放行。

`EndpointManager`

- 生命週期的管理者原始碼連結點我


最後，我們來到了 `pkg/endpointmanager/manager.go`

。`AddEndpoint`

函式標誌著 `Endpoint`

正式被 `cilium-agent`

接管。

```
// pkg/endpointmanager/manager.go
func (mgr *endpointManager) AddEndpoint(ep *endpoint.Endpoint) (err error) {
if ep.ID != 0 {
return fmt.Errorf("Endpoint ID is already set to %d", ep.ID)
}
// ...
// 將 Endpoint "暴露" 給系統
err = mgr.expose(ep)
if err != nil {
return err
}
// ...
// 通知所有 subscribers，一個新的 Endpoint 已經建立
mgr.mutex.RLock()
for s := range mgr.subscribers {
s.EndpointCreated(ep)
}
mgr.mutex.RUnlock()
return nil
}
func (mgr *endpointManager) expose(ep *endpoint.Endpoint) error {
// 1. 分配一個唯一的 Endpoint ID
newID, err := mgr.allocateID(ep.ID)
if err != nil {
return err
}
mgr.mutex.Lock()
// ...
// 2. 啟動 Endpoint 的主 goroutine
ep.Start(newID)
// 3. 將 Endpoint 加入到各個索引 map 中，以便快速查找
mgr.updateIDReferenceLocked(ep)
mgr.updateReferencesLocked(ep, ep.Identifiers())
mgr.mutex.Unlock()
// ...
return nil
}
```


`AddEndpoint`

這裡是個關鍵分界點，因為它會呼叫 `expose()`

，才算是把 Endpoint 正式納入 `cilium-agent`

的管理。

在 `expose()`

裡，首先會透過 `allocateID`

分配一個 Node 範圍內唯一的 ID。接著呼叫 `ep.Start(newID)`

，啟動一個專門管理這個 Endpoint 狀態的 goroutine。然後再把這個 Endpoint 放進各種查詢用的 map（例如 `id -> endpoint`

、`ip -> endpoint`

），讓 agent 之後能快速找到它。最後，`EndpointManager`

還會廣播一個事件通知其他模組，宣告「一個新的 Endpoint 已經建立」。

換句話說，這一步開始，Endpoint 從一個剛被建構的物件，真正變成 `cilium-agent`

內部可見、可管理的實體。

到這裡為止，我們已經看到控制平面怎麼把一個新的 Endpoint 從無到有建起來，驗證完之後交給 `EndpointManager`

，分到唯一的 ID，還有自己的 goroutine 在跑，正式進入管理流程。

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

0

foxracle

iT邦新手 5 級 ‧
2025-10-10 14:03:14

请教一下：

endpointAPIManager 透过 EndpointCreator 创建 ep 时，ep.ID应该是已经赋值了， 但是在 AddEndpoint 里面 第一个 check 就是 ep.ID != 0，如果有值，直接exit，这样就无法进入后面的 expose()和 通知所有 subscribers，一個新的 Endpoint 已經建立。 请教一下：这个ID到底是在哪里赋值的，谢谢！

-
回應 2
- 檢舉

Hi foxracle,

謝謝你的提問！抱歉我還是 iTHome 新手，所以前幾天都沒辦法回應，剛剛才解好新手任務升級

以下是針對你問題的回答：

ep.ID应该是已经赋值了


沒錯，ep.ID 確實有被賦值，而且值就是 `0`

。為什麼會是 0 呢？這就要提到 Go 的一個基本特性：當你創建 struct 但沒有顯式指定某個欄位的值時，Go 會自動把它初始化成該型別的 zero value。如果這裡不太清楚可以看看這個連結體驗一下。

知道這個概念後，我們來看實際創建 `EndpointChangeRequest`

的程式碼就很清楚了。在 CNI Plugin 裡，確實沒有顯式給 `ID`

賦值，所以 ep.ID 就是 `0`

：

程式碼在這裡


```
// plugins/cilium-cni/cmd/endpoint.go
func (c *defaultEndpointConfiguration) PrepareEndpoint(ipam *models.IPAMResponse) (cmd *CmdState, ep *models.EndpointChangeRequest, err error) {
ep = &models.EndpointChangeRequest{
ContainerID: c.Args.ContainerID,
Labels: models.Labels{},
State: models.EndpointStateWaitingDashForDashIdentity.Pointer(),
Addressing: &models.AddressPair{},
K8sPodName: string(c.CniArgs.K8S_POD_NAME),
K8sNamespace: string(c.CniArgs.K8S_POD_NAMESPACE),
K8sUID: string(c.CniArgs.K8S_POD_UID),
ContainerInterfaceName: c.Args.IfName,
DatapathConfiguration: &models.EndpointDatapathConfiguration{},
// 注意這裡沒有設定 ID！
}
}
```


但是在 AddEndpoint 里面 第一个 check 就是 ep.ID != 0，如果有值，直接exit


對，這個設計其實很有道理。既然我們知道 `ep.ID = 0`

是因為一開始沒有顯式賦值，那如果走到 `AddEndpoint()`

時發現 ID 不是 0，就代表有人在中途偷偷設了 ID，這是不被允許的。

能走到 `AddEndpoint()`

就表示這是要創建全新的 Endpoint，按照設計 ep.ID 應該要是 `0`

，讓 Cilium 來負責分配真正的 ID。所以 `ep.ID != 0`

就直接 exit 是合理的保護機制。

这个ID到底是在哪里赋值的


真正的 ID 分配是在這個流程：

- allocateID() 分配新的 ID
- ep.Start(newID) 才真正把 ID 設進去

推薦你點開上面兩個連結，然後看裡面的註解！我覺得註解寫得蠻不錯的

假設我們起點是 AddEndpoint()，那整個流程是：

```
AddEndpoint()
-> 檢查 ep.ID != 0？如果不是 0 就報錯退出
-> endpointManager.expose()
-> allocateID() 分配新 ID
-> ep.Start(newID) 真正設置 ID
```


希望這樣解釋有幫到你！

非常感谢答疑解惑。 重新从cmd的Add开始梳理ep的整个来龙去脉，确实一直没有被赋值，所以默认就是0。从代码注释和debug日志这边也确认了确实是0

```
// e.ID assigned here
err = m.endpointManager.AddEndpoint(ep)
```


```
level=debug msg="PUT /endpoint/{id} request" endpoint="{Addressing:0xc00308f8c0 ContainerID:4b27ee4b2a3a3d1bdc87eef997df6aeb42453d6d688f16546d269640e623d74c ContainerInterfaceName:eth0 ContainerName: DatapathConfiguration:0xc00095be00 DatapathMapID:0 DisableLegacyIdentifiers:false DockerEndpointID: DockerNetworkID: HostMac:8a:c1:05:74:00:a3 ID:0 InterfaceIndex:15 InterfaceName:lxc89cc8f257562 K8sNamespace:netobserv-test K8sPodName:client K8sUID:49cf9fee-0836-4f8b-87a5-97143b8704ea Labels:[] Mac:36:1a:5a:ec:92:89 NetnsCookie:24577 ParentInterfaceIndex:0 Pid:0 PolicyEnabled:false Properties:map[] State:0xc00095be10 SyncBuildEndpoint:true}" subsys=daemon
```


关于ep.ID还有一个疑惑就是下面这块代码，在解析完endpoint parameters之后，立马确认ep.ID是否重复，这个逻辑就很奇怪，还是说有其他可能路径走到这里。什么情况下会存在ID为0的endpoint，而且还是已经被endpointManager接管的endpoint呢？

```
ep, err := m.endpointCreator.NewEndpointFromChangeModel(ctx, epTemplate)
oldEp := m.endpointManager.LookupCiliumID(ep.ID)
if oldEp != nil {
return invalidDataError(ep, fmt.Errorf("endpoint ID %d already exists", ep.ID))
}
```
