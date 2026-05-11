# [DAY13] 探索 Cilium Pod to Pod Datapath (2) eBPF 走捷徑直接送封包到目的 Pod 面前

> 原文連結：https://ithelp.ithome.com.tw/articles/10388758

---

DAY
13

Cloud Native
### 30 天深入淺出 Cilium ：從入門到實戰系列 第
13 篇

##
[Day 13] 探索 Cilium Pod to Pod Datapath (2) eBPF 走捷徑直接送封包到目的 Pod 面前


在昨天，我們從 client pod ping server pod 簡單看了一下 ARP 怎麼被偽造的。

接下來就可以來看看真正一個 TCP 封包是經歷了什麼黑魔法，一路抵達目的地的！

前面已經知道 ARP 回應會被偽造了，那現在如果 client pod 要用 TCP 和 IPv4 封包送出請求 (例如：curl http://10.244.2.61) 實際這封包會怎麼被處理呢？

首先封包會到 client Pod host-side veth，並且執行 TC Ingress 上的 BPF 程式，也就是我們前面提到的 `cil_from_container`


在 `cil_from_container`

內 → 判斷是 IPv4 → tail call:

```
// bpf/bpf_lxc.c
__section_entry int cil_from_container(struct __ctx_buff *ctx)
{
__u16 proto = 0;
bool valid_ethertype = validate_ethertype(ctx, &proto);
bpf_clear_meta(ctx);
switch (proto) {
case bpf_htons(ETH_P_IP): // 命中 IPv4
edt_set_aggregate(ctx, LXC_ID);
// tail call 「接力」給 CILIUM_CALL_IPV4_FROM_LXC
return tail_call_internal(ctx, CILIUM_CALL_IPV4_FROM_LXC, NULL);
...
}
}
```


到 **tail_handle_ipv4**（也就是 `CILIUM_CALL_IPV4_FROM_LXC`

的實作）這裡比較複雜，我用文字簡述，這裡先建立一個流程概觀，後面才會來看原始碼實際感受到 BPF 怎麼「可程式化的走捷徑」：

-
**L3/L4 parse & tuple 建立**- 解析 IPv4 header、TCP 5-tuple (src/dst ip, src/dst port, proto)

-
**Per-Packet LB 檢查**- 如果啟用了
`ENABLE_PER_PACKET_LB`

，會呼叫`__per_packet_lb_svc_xlate_4()`

嘗試匹配 Service - 就算打的是 Pod IP，這個檢查仍會跑過，但最後會跳過 LB 邏輯

- 如果啟用了
-
**BPF Conntrack (CT) 查表 / 建立**- 呼叫
`ct_lookup4()`

查`cilium_ct_*_global`

(BPF CT maps)，看看這個 5-tuple 是否已有 state-
`CT_ESTABLISHED`

→ 回程/後續流量可走 fast path - 沒有 → 建立
`CT_NEW`

項目 (`ct_create4()`

)，並附帶一些 flags (例如是否曾被 L7 代理 redirect 等)

-
- 這個階段會透過
`TAIL_CT_LOOKUP4`

串接到`handle_ipv4_from_lxc`

，而不是再一次 tail call

- 呼叫
-
**ipcache 查詢 + egress policy 檢查**- 在
`handle_ipv4_from_lxc`

透過`lookup_ip4_remote_endpoint(ip4->daddr, cluster_id)`

查`cilium_ipcache`

- 把
**dst IP (10.244.2.61)**轉成**security identity** - 並判斷它是不是 local CEP (影響後面 local deliver 還是走 underlay/overlay)

- 把
- 執行 egress policy 檢查 (
`ipv4_policy()`

的 egress path)- 查
`cilium_policy`

map：client identity → server identity 是否允許 - 如果不允許 → 封包 drop
- 如果綁 L7 policy → 設定
`proxy_port`

，redirect 到 proxy(直連 Pod IP 的 case 通常不會)

- 查

- 在
-
**Service LB / NAT (不會命中)**- 我們打的是 Pod IP，不是 ClusterIP/NodePort →
`lb{4,6}`

family (kube-proxy replacement) 不會參與 - DNAT/SNAT 也不會在這條路徑發生( 沒有 NodePort / ExternalIP / egress NAT)

- 我們打的是 Pod IP，不是 ClusterIP/NodePort →
-
**路由決策 (local vs remote)**-
**Local CEP (我們的 case：server Pod 在同 Node)**-
查

`endpoint_map`

/`cilium_lxc`

：把 server Pod 對應到目標 CEP 的 ifindex / LXC ID -
執行

`ipv4_local_delivery()`

，它不直接 redirect，而是做一個關鍵 tail call：`ret = tail_call_policy(ctx, ep->lxc_id)`

- 透過
`cilium_call_policy`

map，跳到 server 端的`cil_lxc_policy()`

(也就是目標 Pod 的 ingress policy 程式)

- 透過

-
-
**Remote CEP (在不同 Node)**- 會走 host 路徑 (bpf_host.c)，做 FIB lookup / overlay (例如
`cilium_vxlan`

) 或 underlay 發送 (`cil_to_netdev`

) - 但這不是我們這條路線

- 會走 host 路徑 (bpf_host.c)，做 FIB lookup / overlay (例如

-

以我們前面討論到的，經歷一段複雜的邏輯，最終會走到的 BPF Program 是 `ipv4_local_delivery`

:

原始碼連結點我


```
// bpf/bpf_lxc.c
static __always_inline int handle_ipv4_from_lxc(struct __ctx_buff *ctx, __u32 *dst_sec_identity,
__s8 *ext_err)
// ... 略 ...
ep = __lookup_ip4_endpoint(daddr); // 找到 server Pod 的 endpoint
if (ep) {
// ... 略 ...
return ipv4_local_delivery(ctx, ETH_HLEN, SECLABEL_IPV4,
MARK_MAGIC_IDENTITY, ip4,
ep, METRIC_EGRESS, from_l7lb,
false, 0);
}
```


`ipv4_local_delivery`

具體實作是寫在 bpf/lib/local_delivery.h

原始碼連結點我


```
// bpf/lib/local_delivery.h
static __always_inline int ipv4_local_delivery(struct __ctx_buff *ctx, int l3_off,
__u32 seclabel, __u32 magic,
struct iphdr *ip4,
const struct endpoint_info *ep,
__u8 direction, bool from_host,
bool from_tunnel, __u32 cluster_id)
{
mac_t router_mac = ep->node_mac;
mac_t lxc_mac = ep->mac;
int ret;
cilium_dbg(ctx, DBG_LOCAL_DELIVERY, ep->lxc_id, seclabel);
ret = ipv4_l3(ctx, l3_off, (__u8 *)&router_mac, (__u8 *)&lxc_mac, ip4);
if (ret != CTX_ACT_OK)
return ret;
return local_delivery(ctx, seclabel, magic, ep, direction, from_host,
from_tunnel, cluster_id);
}
// 跳來 local_delivery 後，最重要就是，最後面的 tail_call_policy(ctx, ep->lxc_id)
local_delivery(struct __ctx_buff *ctx, __u32 seclabel,
__u32 magic __maybe_unused,
const struct endpoint_info *ep __maybe_unused,
__u8 direction __maybe_unused,
bool from_host __maybe_unused,
bool from_tunnel __maybe_unused, __u32 cluster_id __maybe_unused)
{
/* Jumps to destination pod's BPF program to enforce ingress policies. */
local_delivery_fill_meta(ctx, seclabel, true, from_host, from_tunnel, cluster_id);
// 關鍵 tail call!!!
return tail_call_policy(ctx, ep->lxc_id); // ep->lxc_id 是 server Pod 的 endpoint ID
#endif
}
```


這裡大概做的事情是：

-
**重新貼標籤**：拿到封包後，把收件人 (目的 MAC) 和寄件人 (來源 MAC) 的標籤換掉，確保能送到正確的 Pod -
**直接送達**：它不把包裹交給大樓的收發室 (Linux Network Stack)，而是直接搭電梯（`tail_call_policy`

）衝到收件 Pod 的門口，把包裹交給門口的警衛 (Ingress BPF 程式) 做安全檢查 (Policy Enforcement)

進到 server Pod 的 TC ingress，理所當然也是會從熟悉的 `cil_to_container`

來處理

```
// bpf/bpf_lxc.c
__section_entry int cil_to_container(struct __ctx_buff *ctx)
{
enum trace_point trace = TRACE_FROM_STACK;
__u32 magic, identity = 0;
__u32 sec_label = SECLABEL;
__s8 ext_err = 0;
__u16 proto;
int ret;
// 從 metadata 取得 client Pod 的 Identity
magic = inherit_identity_from_host(ctx, &identity);
switch (proto) {
case bpf_htons(ETH_P_IP):
sec_label = SECLABEL_IPV4;
ctx_store_meta(ctx, CB_SRC_LABEL, identity); // 存放 client Identity
ret = tail_call_internal(ctx, CILIUM_CALL_IPV4_CT_INGRESS, &ext_err);
break;
// ...
}
}
```


```
// bpf/bpf_lxc.c
__declare_tail(CILIUM_CALL_IPV4_TO_ENDPOINT) int tail_ipv4_to_endpoint(
struct __ctx_buff *ctx)
{
__u32 src_sec_identity = ctx_load_and_clear_meta(ctx, CB_SRC_LABEL);
void *data, *data_end;
struct iphdr *ip4;
__u16 proxy_port = 0;
__s8 ext_err = 0;
int ret;
// 執行 server Pod 的 Ingress Policy 檢查
ret = ipv4_policy(
ctx, ip4, src_sec_identity, NULL, &ext_err, &proxy_port, false);
// 如果 ret == CTX_ACT_OK，封包成功抵達 server Pod！
return ret;
}
```


`CTX_ACT_OK`

就像是 BPF 對 kernel 說：「我檢查完了，這封包沒問題，你接手處理吧！」

此時封包會，離開 TC Hook，進入的 Linux network stack，接著封包會送到 server Pod 的 Network Namspace，由於之前 `ipv4_local_delivery`

已經修改了 L2 header，封包會被正確路由到 server Pod 的 `veth`

，然後進行 TCP/IP 處理，一直到應用程式接收，以我們這裡為例我們是使用 NGINX，所以 NGINX 就會取得封包並處理

