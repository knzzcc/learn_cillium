# [DAY10] 當一個 Pod 被建立時 Cilium 在背後做了什麼？(4) BPF 物件與 Datapath 的成形

> 原文連結：https://ithelp.ithome.com.tw/articles/10386695

---

DAY
10

Cloud Native
### 30 天深入淺出 Cilium ：從入門到實戰系列 第
10 篇

##
[Day 10] 當一個 Pod 被建立時 Cilium 在背後做了什麼？(4) BPF 物件與 Datapath 的成形


Day 7 到 Day 9，我們一路走過了 Pod 建立後 Cilium 的幕後流程：

-
**Day 7**聚焦在 CNI Plugin 怎麼把 Pod 的網路請求丟給 cilium-agent -
**Day 8**追蹤了 Endpoint 在 agent 內的誕生過程，如何被「收編」成一個受管理的實體 -
**Day 9**我們看到了 Identity 的登場，Endpoint 在獲得 Identity 之後，觸發第一次 Regeneration

還記得我之前用一句話簡單說明 Regeneration **就是將期望的網路狀態同步到實際的 BPF Datapath 的過程**，所以 Regeneration 是一個非常關鍵的步驟，我們都知道 Cilium 是一個 BPF-based 的產品，那理所當然一個 Pod 被建立時肯定有相關的 BPF Program, BPF Maps… 等需要被編譯、載入、更新等等，這些 BPF 物件是 Datapath 的成形關鍵，沒有 Cilium 把這些抽象的東西 (Identity, Policy… 等) 編譯成 BPF Program 並載入到 Kernel 執行，那我們就無法凌駕在 BPF 之上實現可程式化的 Datapath。

今天的文章，就是要深入探討當一個 Pod 被建立時，Regeneration 到底具體怎麽執行才得以賦予 Cilium 處理 BPF 物件讓 Datapath 成形！

實際上關於 Regeneration 被觸發，有幾個比較細節的重要條件和注意事項，推薦感興趣的讀者們也可以去看一下原始碼註解，以下我把內容白話解釋：

-
**只有在 Endpoint ID 已被分配後，才會觸發 Regeneration**- Endpoint 在被創建時，先有一個基本的物件 (帶著 Pod 的 IP、metadata) 等
- 但是 Cilium 不會立刻去生成 Datapath (BPF Program、maps)，因為還缺一個關鍵：
`Endpoint ID`

- Endpoint ID 是本地節點內唯一的識別碼，它會被用來決定
**state 目錄的路徑**（像`/var/run/cilium/state/123/`

）以及部分 map key

-
**Identity 提前完成 ≠ 立刻 Regeneration**- 假設 Identity 已經先分配好了（透過 K8s Label → Identity Allocator），但 Endpoint ID 還沒出來，在這種情況下，Cilium 會暫時延遲，不去立刻 Regenerate，而是等到
`endpointmanager.AddEndpoint()`

時才觸發

- 假設 Identity 已經先分配好了（透過 K8s Label → Identity Allocator），但 Endpoint ID 還沒出來，在這種情況下，Cilium 會暫時延遲，不去立刻 Regenerate，而是等到
-
**Identity 還沒分配 → 等 controller 來補**- 如果 Endpoint ID 已經有了，但 Identity 還沒分配（可能還在透過 kvstore 或 CRD 解析），那麼
`identityLabelsChanged()`

這個 controller 就會在 Identity 確定後觸發 Regeneration

- 如果 Endpoint ID 已經有了，但 Identity 還沒分配（可能還在透過 kvstore 或 CRD 解析），那麼

我們來看看程式碼：

原始碼連結點我


```
// pkg/endpoint/endpoint.go
// setRegenerateStateLocked 嘗試把 Endpoint 的狀態切換到「等待 Regeneration」。
// 如果回傳 true，表示在釋放 Endpoint 的 lock 之後，應該呼叫 e.Regenerate()。
readyToRegenerate := e.setRegenerateStateLocked(...)
// ... 略 ...
e.unlock()
if readyToRegenerate {
e.Regenerate(regenMetadata) // 觸發 Regeneration
}
```


這邊 `setRegenerateStateLocked`

會想把 Endpoint 狀態切換成 `StateWaitingToRegenerate`

(就是 Day 8 中提到 Endpoint Lifecycle 的 `wating-to-regnerate`

狀態) ，如果一切條件允許然後狀態被切換成功，Cilium 就會 release endpoint lock 然後立刻呼叫 `Regenerate()`


而實際再去看 `Regnerate()`

的原始碼，會發現它其實不會自己去動 BPF，而是把事件丟到 `eventQueue`

：

原始碼連結點我


```
// pkg/endpoint/policy.go
// Regenerate 負責觸發 Endpoint 的 Regenerate 流程
// 注意：它不會自己去編譯 BPF，而是把「Regenerate 任務」丟進 eventQueue
func (e *Endpoint) Regenerate(regenMetadata *regeneration.ExternalRegenerationMetadata) <-chan bool {
// ... 略 .... 準備 context
var ctx context.Context
// 封裝成一個 EndpointRegenerationEvent
epEvent := eventqueue.NewEvent(&EndpointRegenerationEvent{
regenContext: ParseExternalRegenerationMetadata(ctx, e.getLogger(), cancel, regenMetadata),
ep: e,
})
// 將事件丟進 Endpoint 的 eventQueue
resChan, err := e.eventQueue.Enqueue(epEvent)
if err != nil {
cancel()
close(done)
return done
}
// 啟動 goroutine 等待結果
go func() {
// .... 細節略 ...
// 回報結果
done <- buildSuccess
close(done)
}()
return done
}
```


Event 被拿出來消費後，後面最關鍵的是還會去 call 同一個檔案內的 `regenerate()`

，我們來看一下程式碼

原始碼連結點我


```
// pkg/endpoint/policy.go
// regenerate 執行 Endpoint 的 Regeneration 流程
func (e *Endpoint) regenerate(ctx *regenerationContext) (retErr error) {
// 1. 嘗試取得 build lock，避免多個 build 同時進行
e.buildMutex.Lock()
defer e.buildMutex.Unlock()
if err := e.lockAlive(); err != nil {
return err
}
// 2. 設定狀態為 Regenerating
if e.getState() != StateWaitingForIdentity &&
!e.BuilderSetStateLocked(StateRegenerating, "Regenerating endpoint") {
e.unlock()
return fmt.Errorf("Skipping build due to invalid state: %s", e.state)
}
e.unlock()
// 3. 準備 temporary directory
origDir := e.StateDirectoryPath()
tmpDir := e.NextDirectoryPath()
if err := os.RemoveAll(tmpDir); err != nil && !os.IsNotExist(err) {
return fmt.Errorf("unable to remove old tmpDir: %w", err)
}
if err := os.MkdirAll(tmpDir, 0777); err != nil {
return fmt.Errorf("Failed to create tmpDir: %w", err)
}
// 確保最後一定清掉 tmpDir（不論成功或失敗）
defer func() {
if err := e.lockAlive(); err == nil {
e.removeDirectory(tmpDir)
e.BuilderSetStateLocked(StateReady, "Regeneration complete")
e.unlock()
}
}()
// 4. 呼叫 regenerateBPF: 核心 → 產生 & attach BPF 程式
revision, err := e.regenerateBPF(ctx)
if err != nil {
// 如果失敗，把 tmpDir 改成 failDir 保存
failDir := e.FailedDirectoryPath()
e.removeDirectory(failDir)
os.Rename(tmpDir, failDir)
return err
}
// 5. 成功的話，更新為新的 realized state
return e.updateRealizedState(&ctx.Stats, origDir, revision)
}
```


以上面的程式碼來說，業務邏輯大致上是：

-
**狀態轉換**：- 將 Endpoint 設成
`StateRegenerating`

，避免和其他流程衝突

- 將 Endpoint 設成
-
**暫存目錄 tmpDir**：- 先準備一個乾淨的 tmpDir
- 成功編譯完成後才替換成正式的 state 目錄
- 失敗則把 tmpDir 移去 failDir，方便後續 debug

-
**很重要的步驟，呼叫**`regenerateBPF`


我們接著來看一下 `regenerateBPF()`

的程式碼

簡單一句話說明 `regenerateBPF()`

在幹嘛：就是在「重新安裝 / 更新 Endpoint 對應的 BPF 程式與 maps，確保它的 policy、Conntrack state、DNS、proxy 全部都正確同步」：

原始碼連結點我


```
// pkg/endpoint/bpf.go
// regenerateBPF 是 Endpoint datapath programming 的核心
func (e *Endpoint) regenerateBPF(ctx *regenerationContext) (uint64, error) {
// 1. 等 datapath 初始化 & 拿 compilation lock
<-e.orchestrator.DatapathInitialized()
e.compilationLock.RLock()
defer e.compilationLock.RUnlock()
// 2. Policy 計算 (pre-compilation steps)
// - regeneratePolicy() 計算 SelectorPolicy → DistillPolicy → EndpointPolicy
if err := e.runPreCompilationSteps(ctx); err != nil {
return 0, err
}
// 3. 判斷是否需要 Regeneration (hash endpoint config)
hash, err := e.orchestrator.EndpointHash(e)
if err != nil {
return 0, err
}
if hash != e.bpfHeaderfileHash {
ctx.datapathRegenerationContext.regenerationLevel = regeneration.RegenerateWithDatapath
}
// 4. 如果需要，寫 header file
if ctx.datapathRegenerationContext.regenerationLevel >= regeneration.RegenerateWithDatapath {
if err := e.writeHeaderfile(ctx.datapathRegenerationContext.nextDir); err != nil {
return 0, err
}
}
// 5. 真正執行 datapath reload
// - 呼叫 realizeBPFState() → ReloadDatapath() attach BPF
if err := e.realizeBPFState(ctx); err != nil {
return 0, err
}
// 6. 更新 PolicyMap、EndpointMap (確保 BPF maps 同步)
if err := e.policyMapSync(ctx.datapathRegenerationContext.policyMapDump, &ctx.Stats); err != nil {
return 0, err
}
return ctx.datapathRegenerationContext.epInfoCache.revision, nil
}
```


我們白話總結一下 `regenerateBPF()`

在做什麼事情：

- 先算 policy、寫 Endpoint header file
- 判斷要不要重新編譯
- 如果要 → clang 編譯 BPF →
`.o`

→ load 進 kernel

- 如果要 → clang 編譯 BPF →
- 更新 maps / proxy
- 確保 DNS & Conntrack 狀態一致

所以我們現在已經知道，`regenerateBPF()`

的角色就是判斷是否需要重新安裝或更新 BPF program 與相關的 Maps。而在這個流程中，程式最終會呼叫到一個關鍵函式 — `ReloadDatapath()`

，這也是我們這篇文章的後半段要來深入理解的部分

在剛剛的 `regenerateBPF()`

程式碼中，讀者們是否有注意到這一段程式碼呢？

```
func (e *Endpoint) regenerateBPF(...)
// ... 略 ...
if err := e.writeHeaderfile(datapathRegenCtxt.nextDir); err != nil {
return 0, fmt.Errorf("write endpoint header file: %w", err)
}
```


其實，Cilium 的 BPF 程式不是「一次編好，大家共用」，而是**每個 Endpoint 都量身打造**的。

這種訂製是透過在編譯前動態生成一個 C 語言 Header File 來實現的

而這動態產生的 header file 會命名為 `ep_config.h`

，實際 `ep_config.h`

裡面的內容大概會長這樣：

```
#define ENDPOINT_ID 123
#define SECLABEL 16345
#define ....
```


在目前的 Cilium 架構中，這個職責由 `pkg/datapath/loader`

套件來協調

原始碼連結點我


下面是 `ReloadDatapath`

的程式碼 Snippet，一句話來解釋 `ReloadDatapath`

在幹嘛：就是幫指定的 Endpoint 把對應的 BPF program 裝進 kernel：

```
// pkg/datapath/loader/loader.go
// ReloadDatapath reloads the BPF datapath programs for the specified endpoint.
func (l *loader) ReloadDatapath(ctx context.Context, ep datapath.Endpoint, lnc *datapath.LocalNodeConfiguration, stats *metrics.SpanStat) (string, error) {
// ...
spec, hash, err := l.templateCache.fetchOrCompile(ctx, lnc, ep, &dirs, stats)
// ...
}
```


上面的關鍵地方的就是呼叫 `fetchOrCompile()`

，`fetchOrCompile`

會：

-
**Check cache**：先確認這個`EndpointConfiguration`

有沒有對應的 pre-compiled BPF object (`.o`

) 已經存在 cache 裡 -
**Compile or reuse**：- 如果
**cache hit**，就直接複製一份現有的 ELF object，避免重複 compile - 如果
**cache miss**，就會呼叫`o.build()`

，把**generic BPF source (例如**跟`bpf_lxc.c`

)`writeHeaderfile()`

產生的**endpoint-specific header (**丟給`ep_config.h`

)**clang/LLVM**compile，產生新的`.o`

file

- 如果
-
**Return result**：最後回傳一個`ebpf.CollectionSpec`

，這是一個 BPF ELF 的完整描述，包含所有準備要 load 進 kernel 的 program 和 map 定義

感興趣的讀者可以去看一下

`fetchOrCompile()`

的原始碼喔，由於篇幅關係就不特別展開介紹

而我們系列文探討的情境是 「在一個全新的環境中建立 Pod 時，Cilium 在背後做的事情」。

理所當然這時候 cache 不會命中，所以流程會產生 Endpoint 專屬 header 檔案 (ep_config.h)，接著在 fetchOrCompile() 裡呼叫 clang/LLVM 把這個 header 與通用的 BPF source code (例如 bpf_lxc.c) 一起編譯，產生新的 BPF object (.o)

`bpf_lxc.c`

前面提到了 Endpoint 會有自己專屬的 Header file，有了包含 Endpoint 特定資訊的 header file，接下來就需要一個 BPF 程式的「範本」。`bpf/bpf_lxc.c`

就是這個範本，它定義了 attach 到 Pod veth 上的 BPF 程式的**通用邏輯**

讓我們看看它是如何使用上面生成的 `#define`

Marco 的：

原始碼連結點我


```
// bpf/bpf_lxc.c
// 1. 引入動態生成的 header file
#include <bpf/config/endpoint.h>
#include <bpf/config/lxc.h>
// ... 引入其他函式庫 ...
// 2. 在程式邏輯中使用 marco
static __always_inline int handle_ipv4_from_lxc(struct __ctx_buff *ctx, __u32 *dst_sec_identity,
__s8 *ext_err)
{
// ...
// 3. 進行 Policy 檢查時，傳入本地 Endpoint 的 Security Identity (SECLABEL)
verdict = policy_can_egress4(ctx, &cilium_policy_v2, tuple, l4_off, SECLABEL_IPV4,
*dst_sec_identity, &policy_match_type, &audited,
ext_err, &proxy_port);
// ...
}
__section_entry
int cil_from_container(struct __ctx_buff *ctx)
{
__u16 proto = 0;
__u32 sec_label = SECLABEL;
__s8 ext_err = 0;
// ...
}
```


`bpf_lxc.c`

透過 `#include`

將 `ep_config.h`

的內容合併進來

當 C 預處理器執行時，`SECLABEL`

就會被直接替換成 `cilium-agent`

寫入的具體數字（例如 `16345`

）。

到這裡，一個為特定 Endpoint 量身打造的、包含其 Identity 資訊的 BPF C Code 就準備好了

還記得我們在 Day 4 有講解 eBPF 的運作流程嗎？現在有了完整的 C 程式碼，就可以來編譯了，cilium-agent 會呼叫 clang/LLVM 來執行編譯，複習一下指令：

```
clang -O2 -target bpf -c bpf_lxc.c -I/path/to/headers -o /var/run/cilium/state/123/bpf_lxc.o
```


-
`target bpf`

：告訴`clang`

將程式碼編譯成 BPF bytecode -
`c bpf_lxc.c`

：指定要編譯的檔案 -
`I/path/to/headers`

：包含動態生成的`ep_config.h`

所在的目錄以及 Cilium BPF 函式庫的 header file 路徑 -
`o .../bpf_lxc.o`

：指定輸出的 ELF 檔要放在哪

接著就是進行 Load 和 Attach 的動作，但是 cilium-agent 絕對不是用 `bpftool`

這個工具來 Load / Attach，而是使用 ebpf-go 函式庫 ，具體 cilium-agent 是這麼做的真的感興趣的可以去看原始碼 ( `pkg/datapath/loader/complie.go`

)，我這裡就不展開原始碼介紹，用文字白話簡單解釋：

-
**解析 ELF 物件檔**：cilium-agent 透過解析`bpf_lxc.o`

檔案，讀取其中的 BPF bytecode 和 BPF Map 的定義 -
**建立與更新 BPF Maps**：Agent 會根據 ELF 檔案中的定義，透過`bpf()`

system call 來建立或更新 Endpoint 所需的 BPF Maps -
**載入 BPF 程式**：Agent 一樣使用 ebpf-go 將 BPF Bytecode 載入到 Kernel 中（底層肯定是使用`bpf()`

system call 的`BPF_PROG_LOAD`

command) -
**Attach 到 network interface**：這是最後的臨門一腳。cilium-agent 會找到代表 Pod 的 veth pair 在**Host 端的 interface**（例如`lxc123`

），然後使**Go 語言的 netlink 封裝 (或 ebpf-go library)**，直接呼叫 Linux kernel 的**netlink syscall**的方法將 BPF 程式掛載到該介面的 ingress (或 egress) hook 上

*圖片取自：https://docs.cilium.io/en/stable/_images/cilium-endpoint-lifecycle.png*

最終 Endpoint 會被切換成 `Ready`

狀態，具體切換成 `Ready`

狀態的程式碼其實寫在 `regenerate()`

function 裡面

在 `regenerate()`

裡，Endpoint 狀態的更新是透過 Go 的 defer 機制完成的，

會在 `regenerate()`

這個函式 return 之前最後執行，確保無論 regeneration 成功或失敗，Endpoint 的狀態都會被正確收斂：

原始碼連結點我


```
// pkg/endpoint/policy.go
func (e *Endpoint) regenerate
defer func() {
if err := e.lockAlive(); err != nil {
if retErr == nil {
retErr = err
} else {
e.logDisconnectedMutexAction(err, "after regenerate")
}
return
}
// 不管成功或失敗，都要清理掉暫存資料夾
e.removeDirectory(tmpDir)
// 最後切換到 Ready 狀態
e.BuilderSetStateLocked(
StateReady,
"Completed endpoint regeneration with no pending regeneration requests",
)
e.unlock()
}()
```


換句話說：

-
`regenerate()`

本體會先去呼叫`regenerateBPF()`

、做 tmpDir → origDir 的切換等等 - 等到
`regenerate()`

即將 return 結果（成功 or 失敗）時，Go runtime 會自動觸發所有`defer`

區塊 - 那個時候才會清理 tmpDir，並把 Endpoint 狀態切回 Ready

今天的文章也是一個非常重要的主題，我們都知道 Cilium 是一個 eBPF-based 的產品，今天的這篇文章便深入瞭解 Cilium 怎麼把 BPF 物件產出來一路 attach 到 hook point 形成可程式化的 Datapath：

-
**觸發**：Go 層的狀態變更觸發 Regenerate -
**專屬 Header file**：Go 層將 Endpoint 的上下文資訊寫入動態生成的 Ｃ Header file -
**Compile**：clang/LLVM 將 BPF 範本和標頭檔編譯成特定於 Endpoint 的 BPF 物件檔。 -
**Load**：cilium-agent 解析 ELF 檔，透過 ebpf-go 函式庫將 BPF 程式和 Maps 載入 Kernel -
**Attach**：`tc`

命令 (或 TCX) 將 BPF 程式 attach 到 Pod 的 host-side veth 介面上的 TC hook point，使其正式生效

另外，今天這篇也是「當一個 Pod 被建立時 Cilium 在背後做了什麼？」小小系列文的最後一篇了！接下來我們會來實際探索 Datapath

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
