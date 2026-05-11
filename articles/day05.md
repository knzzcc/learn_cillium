# [DAY05] 正式踏入 Cilium 前，先搞懂 eBPF - BPF Maps 與 Pining 篇

> 原文連結：https://ithelp.ithome.com.tw/articles/10382972

---

DAY
5

在 Day 4 我們已經認識了 eBPF 的概念，當時我們只提到了一些 High Level 的概念、講解了 eBPF 的歷史和 eBPF Program 的生命週期，Day 4 所教學的內容確實也足以讓我們開發和執行 eBPF Program，但是我們會有這些需求：

- 如果需要紀錄資料，要儲存在哪？難道又要依賴回 user space?
- Cilium Agent 或是你的 Application 運行在 User Space ，那在 User Space 怎麼和 Kernel 裡的 BPF 物件互動？

以上這些需求，都可以透過 BPF Map 和 Pining 來滿足！所以 Day 5 我們就來認識 BPF Map 和 Pining

如果對於

實作有興趣的話，在 2025/10/22 16:00 - 17:00 臺北文創大樓 6 樓 DE 會議室 6F 我將有一場演講歡迎來參加 — 「從零打造 eBPF CNI Plugin：揭秘 Cilium 封包處理核心原理」，這場演講會透過我實作的 Cilium Prototype 去理解 Cilium 的運作。

活動連結：https://k8s.ithome.com.tw/2025/session-page/4083

直接以一個情境來舉例，假設我們寫了一個 BPF Program 是要「計數放行幾個封包」，那我們要把 Count 記錄在哪？每一次事件來了，經過 hook point 時執行的程式碼都是獨立發生的 (無狀態)，而 BPF 提供了 11 個 64 bits 暫存器和一個 512 bytes 的 stack space，但一旦程式執行完畢，這些暫存器和 stack 中的狀態就會被銷毀，怎麼可能記住上一次執行發生什麼？進度到哪？

可以參考此文件深入理解 BPF 的 Instruction Set


而在 Day 4 提到 Verifier 時，我們已經知道 eBPF 是會被動態插入到 Kernel 執行，所以也不可以越界存取 Memory 或是分配記憶體，所以也不能妄想說我們把想要紀錄的資料儲存到記憶體後讓 BPF Program 直接存取

所以上面的情境很自然就衍生出「儲存」的需求，於是 Kernel 提供一組資料結構叫 **BPF** **Maps**（hash table、array、LRU hash…），用來在 Kernel 與 User Space 之間、以及在 eBPF 程式的多次調用之間共享資料。

可以參考此文件得知有哪些 BPF Maps 類型

![[Pasted image 20260511164619.png]]

整理一下，BPF Maps 解決以下問題：

- Kernel Space - User Space 通訊
- 多次調用的狀態持久化
- 多個 eBPF Program + User Space 共享資料

這段落會講解 BPF Maps 是怎麼被創建，最後又是怎麼被銷毀的，同時也會講解 Pining 的概念。

還記得我們在 Day 4 提到，在 BPF 這裡不管什麼操作，底層都離不開 `bpf()`

這個 system call 嗎？所以要創建一個 BPF Map 底層就是透過 `bpf(BPF_MAP_CREATE, ...)`

這樣的 system call 而創建的。當**創建成功後會返回一個 File Descriptor (FD)**，所以在 User Space 如果要和新創建的 BPF Map 互動就會透過這個 FD

實際創建起來的感覺是這樣：

`bpf()`

system call 需傳入的參數長這樣 **`bpf**(int command, union bpf_attr *attr, u32 size)`


比較關鍵需要特別講的就是 `union bpf_attr`


```
union bpf_attr attr = {
.map_type = BPF_MAP_TYPE_ARRAY; /* mandatory */
.key_size = sizeof(__u32); /* mandatory */
.value_size = sizeof(__u32); /* mandatory */
.max_entries = 256; /* mandatory */
.map_flags = BPF_F_MMAPABLE;
.map_name = "example_array";
};
```


-
`map_type`

：指定要創建的 Map 類型。這個值決定了核心將使用哪種底層資料結構實現，例如 Hash table 或 arrary。 -
`key_size`

：以 byte 為單位定義鍵的大小 -
`value_size`

：以 byte 為單位定義值的大小 -
`max_entries`

：設定 Map 能夠容納的最大元素數量 -
`map_flags`

：用於指定額外的行為，詳細可以參考此文件- 例如：
`BPF_F_MMAPABLE`

允許 User space 透過`mmap()`

直接映射 Array Map 的記憶體，而`BPF_F_NO_PREALLOC`

則指示 Kernel 不要為 Hash Map 預先配置所有記憶體，以節省資源 。

- 例如：

歡迎參考 Linux Kernel 的文件


關於

`BFP_MAP_CREATE`

可以參考此文件

只是上面提到的東西都是比較底層的操作，其實有一些工具或是函式庫都封裝好那些操作了，例如：bpftool, libbpf

以 `bpftool`

來做示範：

```
# 創建一個名為 my_map 的 hash map，並將其釘選到 /sys/fs/bpf/my_map
sudo bpftool map create /sys/fs/bpf/my_map type hash key 4 value 8 entries 1024 name my_map
```


此命令創建了一個 Key 大小為 4 bytes、value 大小為 8 bytes、最多可容納 1024 個 entries 的 hash map 。

BPF Map 的生命週期管理核心原理在於一個簡單而強大的機制 — 引用計數 (`refcnt`

)，簡單來說它就是一個「**有多少人還在用這個東西**」的計數器。這個機制是 Kernel 內部用於追蹤所有 BPF Objects（包括 Map 和 Program）生命週期的唯一真實來源

你只要記住：`refcnt`

大於 0，Kernel 就會確保其存活。只有當 `refcnt`

降至 0 時，該物件才有資格被 Kernel 回收。

- 所以當我們透過
`bpf(BPF_MAP_CREATE, ...)`

成功創建 Map 後，`refcnt`

會被初始化為 1 - 如果寫了一個 eBPF 程式，程式碼會需要用到此 **Map (例如：前面提到的紀錄放行的封包數量)，**當這個程式載入 kernel、並且通過
**Verifier**驗證後，kernel 會再幫這個 Map 的`refcnt`

+1 - 接著再將 eBPF 程式 Attach 到 hook point 會增加 eBPF 程式本身的
`refcnt`

，所以 eBPF 程式的存活就勁兒維持了它對 BPF Map 的引用 ，**簡單說就是程式還沒死 Map 就死不了**

前面已經提到 `refcnt`

，接下來再來講一個重要的概念 — **Pining，中文我會稱為「釘選」**

Pinning 是指透過在 BPF 檔案系統（BPFFS）中建立一個特殊檔案來為 BPF 物件（Map 或程式）建立一個持久性參考的行為 。這個動作明確地告知核心：「無論是否有程序持有指向它的 FD，都請保持這個物件的活動狀態」

我知道上面文鄒鄒的很難理解，簡單來說 Pining 就是告訴 Kernel：「**即使沒有任何 process 手上還握著 FD（file descriptor），也請讓這個物件繼續活著**」

換句話說，**Pinning 讓 BPF Objects 從「process 資源」變成「系統資源」**。

我們來把 Pining 的重要概念拆解，看一下 Pining 怎實現的：

-
BPFFS (BPF Filesystem)

-
專門給 BPF 物件用的 pseudo filesystem，通常掛載在

`/sys/fs/bpf`

，可以參考以下指令找到目錄`# Unbuntu VM ubuntu@my-instance:~$ mount | grep bpf bpf on /sys/fs/bpf type bpf (rw,nosuid,nodev,noexec,relatime,mode=700) ubuntu@my-instance:~$ sudo ls -l /sys/fs/bpf total 0 lrwxrwxrwx 1 root root 0 Sep 8 01:54 ip -> /sys/fs/bpf/tc/ drwx------ 2 root root 0 Sep 7 03:38 snap drwx------ 3 root root 0 Sep 8 01:54 tc lrwxrwxrwx 1 root root 0 Sep 8 01:54 xdp -> /sys/fs/bpf/tc/`


-
-
要 Pin 一個物件也是用

`bpf(BPF_OBJ_PIN, ...)`

system call- 需要有一個 BPF FD（例如 Map 的 FD），你可以呼叫這個 syscall，把它 pin 成
`/sys/fs/bpf/my_map`

- 這個動作會讓 Map 的
`refcnt`

+1，並建立一個持久 reference -
`BPF_OBJ_PIN`

可以參考這文件

- 需要有一個 BPF FD（例如 Map 的 FD），你可以呼叫這個 syscall，把它 pin 成
-
如果要從 pinned 物件取得新的 FD 會需要用

`bpf(BPF_OBJ_GET, ...)`

system call- 任何其他 process（只要有權限）都可以用這個路徑 (
`/sys/fs/bpf/my_map`

) 來取得新的 FD，指向同一個 Map

- 任何其他 process（只要有權限）都可以用這個路徑 (
-
那要怎麼 Unpining?

- 其實就把
`/sys/fs/bpf/my_map`

這個檔案刪掉（例如`rm`

），就等於少了一個 reference - 如果剛好
`refcnt`

掉到 0，物件就會被釋放

- 其實就把

其實在前面講到持久的關鍵時，就已經提到如何銷毀了，關鍵就是讓 `refcnt`

降至 0，具體方法包括說：

- 銷毀正在使用該物件的物件
- 刪除 BPFFS 裡面的檔案

當 `refcnt`

被降至 0 時，Kernel 會將該 BPF Object 標記為待銷毀，但真正 free 掉，要等一個 RCU grace period，確保沒有 Thread 還在讀這個物件。

Day 5 我們把重點放在 **BPF Maps** 與 **Pinning**。Maps 解決了 eBPF 程式需要保存狀態、以及和 User Space 共享資料的問題；而 Pinning 則讓 BPF 物件不再受限於單一 process 的生命週期，升級成為系統資源。

簡單說，**Maps 提供狀態，Pinning 提供持久性**。這兩個機制讓 eBPF 程式不再只是短暫的 hook，而是能長期穩定運作，並和不同工具模組化協作。

