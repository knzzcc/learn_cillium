# [DAY15] 探索 Cilium Pod to Pod Datapath (4) 跨 Node Datapath 與實際驗證

> 原文連結：https://ithelp.ithome.com.tw/articles/10390038

---

DAY
15

Cloud Native
### 30 天深入淺出 Cilium ：從入門到實戰系列 第
15 篇

##
[Day 15] 探索 Cilium Pod to Pod Datapath (4) 跨 Node Datapath 與實際驗證


在前面的文章中，我們深入探討了「同 Node 的 Pod-to-Pod Datapath」，並在 Day 14 認識了 `cilium_host`

、`cilium_net`

和 `cilium_vxlan`

等關鍵的網路介面。今天我們要正式進入「跨 Node Datapath」的世界，這個主題比同 Node 通信複雜許多，涉及到 VXLAN 隧道、BPF 程式的協調運作。

今天的文章除了會透過深入的原始碼分析還會有實際抓封包來驗證 datapath，我們將會發現裡面的每個重要技術細節，並且完全理解 Cilium 跨 Node 通信的真實機制。

今天的文章會有驗證 Datapath 的環節，本段落提供我驗證時所使用的環境

| 節點 | Node IP | Pod CIDR | cilium_vxlan MAC |
|---|---|---|---|
| worker-1 | 10.13.1.57 | 10.244.2.0/24 | 52:fb:e6:dc:8f:e6 |
| worker-2 | 10.13.1.129 | 10.244.1.0/24 | 96:b8:2d:4f:0d:61 |

| Role | Pod (Name) | Pod IP | Node | Node IP |
|---|---|---|---|---|
| Client (src) | netshoot | 10.244.2.173 | worker-1 | 10.13.1.57 |
| Server (dst) | nginx | 10.244.1.9 | worker-2 | 10.13.1.129 |

- Version: 1.18.1
- Routing mode: tunnel
- Enable-endpoint-routes: false
- 也就是 cilium_host 模式（非 per-endpoint）

- Enable-host-legacy-routing: false (預設值)
- 也就是 BPF Host Routing


在開始前，我們先來看一下每個網卡身上有哪些 BPF 程式，透過 `bpftool net`

我們可以看到各個網路介面上掛載的 BPF 程式，底下為了方便閱讀，輸出內容有稍作簡化：

```
# worker-1
$ bpftool net
tc:
# 物理網卡
ens5(2) tcx/ingress → cil_from_netdev prog_id 4600 # bpf_host.c
ens5(2) tcx/egress → cil_to_netdev prog_id 4602 # bpf_host.c
# Cilium Host 介面
cilium_host(5) tcx/ingress → cil_to_host prog_id 4577 # bpf_host.c
cilium_host(5) tcx/egress → cil_from_host prog_id 4576 # bpf_host.c
# VXLAN 隧道介面
cilium_vxlan(6) tcx/ingress → cil_from_overlay prog_id 4513 # bpf_overlay.c
cilium_vxlan(6) tcx/egress → cil_to_overlay prog_id 4514 # bpf_overlay.c
# Pod veth 介面
lxc3a6f7d4e8c73(26) tcx/ingress → cil_from_container prog_id 4773 # bpf_lxc.c
```


不知道讀者們是否有發現在 Cilium 的 BPF 程式裡，常常會看到一些固定風格的命名，例如：`cil_from_container`

、`cil_to_host`

、`cil_from_overlay`

。這些名字並不是隨便取的，而是有一個明顯的慣例：

-
：前綴代表這支程式是由 Cilium 產生的。`cil_`

-
：用來區分流量方向，`from_*`

/`to_*`

`from`

表示「流量進來這個介面」，`to`

表示「流量要從這個介面送出去」。 -
**介面類型**：後綴則對應不同的掛載點，例如-
`_container`

→ Pod veth (容器流量入口) -
`_host`

→ cilium_host 虛擬介面 (Pod 與 Host 交換流量) -
`_netdev`

→ 實體網卡 (Node 對外的流量) -
`_overlay`

→ VXLAN/Geneve overlay 介面 (跨 Node 封裝/解封裝)

-

讓我們追蹤一個完整的 HTTP 請求流程，並深入了解每個階段的原始碼實現：

當 client Pod 發送請求時，封包首先到達 Pod 的 veth 介面上掛載的 `cil_from_container`

BPF 程式。關鍵的路由決策發生在 `bpf/bpf_lxc.c`

中的 `handle_ipv4_from_lxc()`

函式：

原始碼連結點我


```
// bpf/bpf_lxc.c
static __always_inline int
handle_ipv4_from_lxc(struct __ctx_buff *ctx, __u32 *dst_sec_identity, __s8 *ext_err)
{
struct remote_endpoint_info *info;
/* Determine the destination category for policy fallback. */
info = lookup_ip4_remote_endpoint(ip4->daddr, cluster_id); // 查找 remote endpoint
if (info) {
*dst_sec_identity = info->sec_identity;
skip_tunnel = info->flag_skip_tunnel;
}
// ... Policy 檢查 ...
# if defined(TUNNEL_MODE)
if (ct_state->from_tunnel || !skip_tunnel) {
if (info && info->flag_has_tunnel_ep) { // 確認支援隧道
ret = encap_and_redirect_lxc( // 直接重定向！
ctx, info, SECLABEL_IPV4, *dst_sec_identity,
&trace, bpf_htons(ETH_P_IP));
switch (ret) {
case CTX_ACT_OK:
goto encrypt_to_stack;
// ...
default:
return ret; // 跳過後續處理，不經過 cilium_host！
}
}
}
# endif
}
```


**關鍵實現細節**：

-
`lookup_ip4_remote_endpoint()`

查找目標 IP`10.244.1.9`

的 remote endpoint 資訊 - 發現目標支援隧道 (
`info->flag_has_tunnel_ep`

) - 直接調用
`encap_and_redirect_lxc()`

進行 VXLAN 封裝和重定向 - 函式直接
`return ret`

，完全跳過後續的 cilium_host 處理邏輯

`encap_and_redirect_lxc()`

的實現鏈：

原始碼連結點我


```
// bpf/lib/encap.h
encap_and_redirect_lxc()
→ encap_and_redirect_with_nodeid()
→ __encap_and_redirect_with_nodeid()
→ ctx_redirect(ctx, ENCAP_IFINDEX, 0) // 重定向到 cilium_vxlan
```


其中 `ENCAP_IFINDEX`

的定義來自 `pkg/datapath/tunnel/tunnel.go`

：

原始碼連結點我


```
return dpcfgdef.Map{
"ENCAP_IFINDEX": fmt.Sprintf("%d", tunnelDev.Attrs().Index),
}, nil
```


這裡的 `tunnelDev`

就是 `cilium_vxlan`

設備，因此：

-
**ENCAP_IFINDEX**=`cilium_vxlan`

的 ifindex -
= 直接 redirect 到`ctx_redirect(ctx, ENCAP_IFINDEX, 0)`

`cilium_vxlan`


在 Cilium 中，VXLAN 的封裝參數不是預先靜態設定，而是由 **BPF 程式在封包處理時動態決定**：

原始碼連結點我


```
// bpf/lib/overloadable_skb.h - ctx_set_encap_info4()
struct bpf_tunnel_key key = {};
__u32 key_size = TUNNEL_KEY_WITHOUT_SRC_IP;
// 設定 VNI：預設用 Pod Identity 作為封裝 VNI
key.tunnel_id = get_tunnel_id(seclabel);
// 設定 remote node 的 IPv4 (目的 Node)
key.remote_ipv4 = bpf_ntohl(tunnel_endpoint);
// TTL 設定為預設值
key.tunnel_ttl = IPDEFTTL;
// 最後透過 helper 把封裝資訊寫入 SKB metadata
ret = ctx_set_tunnel_key(ctx, &key, key_size, BPF_F_ZERO_CSUM_TX);
if (unlikely(ret < 0))
return DROP_WRITE_ERROR;
return CTX_ACT_REDIRECT;
```


在 worker-2 上，VXLAN 封包通過 `cilium_vxlan`

介面進入 `cil_from_overlay`

BPF 程式：

原始碼連結點我


```
// bpf/bpf_overlay.c
__section_entry
int cil_from_overlay(struct __ctx_buff *ctx)
{
__u32 src_sec_identity = 0;
__s8 ext_err = 0;
bool decrypted = false;
__u16 proto;
int ret;
// VXLAN 隧道 key 解析
if (!decrypted) {
struct bpf_tunnel_key key = {};
ret = get_tunnel_key(ctx, &key); // 提取 tunnel 資訊
src_sec_identity = get_id_from_tunnel_id(key.tunnel_id, proto); // Identity 識別
ctx_store_meta(ctx, CB_SRC_LABEL, src_sec_identity); // 儲存來源 Identity
}
// 根據內層協議類型跳轉
switch (proto) {
case bpf_htons(ETH_P_IP):
// 關鍵！跳轉到專門的 IPv4 處理程式
ret = tail_call_internal(ctx, CILIUM_CALL_IPV4_FROM_OVERLAY, &ext_err);
break;
}
}
```


這邊的核心就是去調用 `handle_ipv4()`


```
// bpf/bpf_overlay.c
__declare_tail(CILIUM_CALL_IPV4_FROM_OVERLAY)
int tail_handle_ipv4(struct __ctx_buff *ctx)
{
// ... 略 ...
ret = handle_ipv4(ctx, &src_sec_identity, &ext_err); // 調用核心處理函式
if (IS_ERR(ret))
return send_drop_notify_error_ext(ctx, src_sec_identity, ret, ext_err,
METRIC_INGRESS);
return ret;
}
```


接著通過 `handle_ipv4()`

進行路由決策，`lookup_ip4_endpoint()`

找到目標 server Pod 後，直接調用 `ipv4_local_delivery()`

將封包交付到目標 server Pod，同樣跳過 cilium_host。

原始碼連結點我


```
// bpf/bpf_overlay.c - handle_ipv4()
static __always_inline int handle_ipv4(struct __ctx_buff *ctx,
__u32 *identity,
__s8 *ext_err)
{
struct endpoint_info *ep;
// ... 略 ...
/* Deliver to local (non-host) endpoint: */
ep = lookup_ip4_endpoint(ip4); // 查找目標 IP (10.244.1.9) 的本地 endpoint
if (ep && !(ep->flags & ENDPOINT_MASK_HOST_DELIVERY))
// 送去 server Pod
return ipv4_local_delivery(ctx, ETH_HLEN, *identity, MARK_MAGIC_IDENTITY,
ip4, ep, METRIC_INGRESS, false, true, 0); // from_host=false from_tunnel=true
// ... 略 ...
}
```


既然跨節點流量不經過 `cilium_host`

，那它的作用是什麼？

`bpf_host.c`

中的 `handle_ipv4_cont()`

函式只在特定條件下執行跨節點路由：

```
// bpf/bpf_host.c - handle_ipv4_cont()
static __always_inline int
handle_ipv4_cont(struct __ctx_buff *ctx, __u32 secctx, const bool from_host,
__s8 *ext_err)
{
/* Below remainder is only relevant when traffic is pushed via cilium_host.
* For traffic coming from external, we're done here.
*/
if (!from_host)
return CTX_ACT_OK;
// 只有當 from_host=true 時才繼續處理跨節點邏輯
info = lookup_ip4_remote_endpoint(ip4->daddr, 0);
if (info && info->flag_has_tunnel_ep) {
return encap_and_redirect_with_nodeid(ctx, info, secctx, ...);
}
}
```


**cilium_host 主要處理**：

-
**Host-to-Pod 通信**：從 Node 本身發起的流量 -
**L7 Proxy 相關流量**：需要經過 Envoy proxy 的流量 -
**NodePort/LoadBalancer 服務流量**：Kubernetes 服務相關流量 -
**Cilium Health Check 流量**：Node 間健康檢查 -
**特定服務流量**：需要經過 host network stack 的流量

**不處理**：

- 從 Pod 發出的一般跨 Node Pod-to-Pod 流量（包括回程流量）

這裡我會開很多 terminal 並使用 `tcpdump`

來抓封包，來證明封包實際會經過哪些 netdev

在前面我們提到其實跨 Node Pod-to-Pod 不會經過 `cilium_host`

，這裡我們實際抓包，照理來說是不會抓到任何 client Pod (`10.244.2.173`

) 或是 server Pod (`10.244.1.9`

) 相關的封包

```
# worker-1
# 抓 cilium_host 封包
tcpdump -i cilium_host -n
```


```
# Client Pod 向 Server Pod 發出請求
kubectl exec client -- curl -s http://10.244.1.9
```


實際的抓到的封包如下：

```
# worker-1
$ tcpdump -i cilium_host -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on cilium_host, link-type EN10MB (Ethernet), snapshot length 262144 bytes
17:23:43.207096 IP 10.244.1.36.4240 > 10.244.2.161.43926: Flags [.], ack 1164516173, win 486, options [nop,nop,TS val 2241868818 ecr 2531692031], length 0
17:23:43.373819 IP 10.244.1.36.4240 > 10.244.2.161.43926: Flags [.], ack 1, win 486, options [nop,nop,TS val 2241868985 ecr 2531692031], length 0
...
```


如上輸出，你會看到**沒有任何流量**是關於 client Pod (`10.244.2.173`

) 或是 server Pod (`10.244.1.9`

)

反而你看到的是：

-
`10.244.1.36`

(port 4240) : 其實是 cilium-health 的流量，詳細可以看這份官方文件 -
`10.244.2.161`

: 就是 cilium_host

至於 worker-2 的視角，是否會在「接收」封包時經過 cilium_host 呢？事實上我們也可以在 worker-2 上抓 `cilium_host`

的封包：

```
# worker-2
# 抓 cilium_host 封包
tcpdump -i cilium_host -n
```


為了避免篇幅過長，這裡直接說結論：**在接收的路徑上不會經過 cilium_host**：

- worker-2 作為接收端，在
`cilium_host`

沒有抓到 client Pod (`10.244.2.173`

) 或是 server Pod (`10.244.1.9`

) 相關封包

這次我們來驗證 VXLAN 的流量，根據之前的原始碼分析，會預期跨 Node Pod-to-Pod Datapath：

- 會經過 VXLAN
- 會抓到封裝前的封包 (Inner)
- Node 的網卡會看到封裝後的封包 (Outer)

開兩個 Terminal 在 worker-1 執行以下指令：

```
# worker-1
# 觀察 Pod-to-Pod 的 inner 封包
tcpdump -i cilium_vxlan -n host 10.244.2.173 -c 20
# 觀察 Node-to-Node 的 outer VXLAN 封包 (UDP 8472)
tcpdump -i ens5 -n udp port 8472 and host 10.13.1.129 -vv -c 20
```


開兩個 Terminal 在 worker-2 執行以下指令：

```
# worker-2
# 觀察 Pod-to-Pod 的 inner 封包
tcpdump -i cilium_vxlan -n host 10.244.1.9 -c 20
# 觀察 Node-to-Node 的 outer VXLAN 封包 (UDP 8472)
tcpdump -i ens5 -n udp port 8472 and host 10.13.1.57 -vv -c 20
```


接著發出請求：

```
# Client Pod 向 Server Pod 發出請求
kubectl exec client -- curl -s http://10.244.1.9
```


worker-1 抓到的封包如下，由於輸出比較長我有做一點簡化：

```
# worker-1
$ tcpdump -i cilium_vxlan -n host 10.244.2.173 -c 20
18:12:36.121720 IP 10.244.2.173.56304 > 10.244.1.9.80: Flags [P.], seq 1:75, ack 1, win 488, options [nop,nop,TS val 501951332 ecr 340410406], length 74: HTTP: GET / HTTP/1.1
18:12:36.121846 IP 10.244.1.9.80 > 10.244.2.173.56304: Flags [.], ack 75, win 487, options [nop,nop,TS val 340410407 ecr 501951332], length 0
18:12:36.121992 IP 10.244.1.9.80 > 10.244.2.173.56304: Flags [P.], seq 1:239, ack 75, win 487, options [nop,nop,TS val 340410407 ecr 501951332], length 238: HTTP: HTTP/1.1 200 OK
...
```


可以注意到這裡抓到的封包都是 client Pod (`10.244.2.173`

) 或是 server Pod (`10.244.1.9`

)

```
# worker-1
$ tcpdump -i ens5 -n udp port 8472 and host 10.13.1.129 -vv -c 20
18:12:36.122017 IP (tos 0x0, ttl 64, id 19882, offset 0, flags [none], proto UDP (17), length 102)
10.13.1.57.42553 > 10.13.1.129.8472: [no cksum] OTV, flags [I] (0x08), overlay 0, instance 26328
IP (tos 0x0, ttl 64, id 34353, offset 0, flags [DF], proto TCP (6), length 52)
10.244.2.173.56304 > 10.244.1.9.80: Flags [.], cksum 0x547a (correct), seq 75, ack 239, win 487, options [nop,nop,TS val 501951333 ecr 340410407], length 0
18:12:36.122042 IP (tos 0x0, ttl 64, id 56491, offset 0, flags [none], proto UDP (17), length 717)
10.13.1.129.53433 > 10.13.1.57.8472: [no cksum] OTV, flags [I] (0x08), overlay 0, instance 9764
IP (tos 0x0, ttl 64, id 20240, offset 0, flags [DF], proto TCP (6), length 667)
10.244.1.9.80 > 10.244.2.173.56304: Flags [P.], cksum 0xbd87 (correct), seq 239:854, ack 75, win 487, options [nop,nop,TS val 340410407 ecr 501951332], length 615: HTTP
...
```


而上方抓 Node 的網卡也可以注意到是已經封裝過後的封包

**Outer 顯示的 src/dst IP 分別是**：

- worker-1:
`10.13.1.57`

- worker-2:
`10.13.1.129`


**Inner 顯示的 src/dst IP 分別是**：

- client Pod:
`10.244.2.173`

- server Pod:
`10.244.1.9`


至於 worker-2 其實概念都是一樣，我將錄到的封包簡單紀錄上來，但不會重複解釋

```
# worker-2
$ tcpdump -i cilium_vxlan -n host 10.244.1.9 -c 20
18:12:36.121957 IP 10.244.1.9.80 > 10.244.2.173.56304: Flags [P.], seq 1:239, ack 75, win 487, options [nop,nop,TS val 340410407 ecr 501951332], length 238: HTTP: HTTP/1.1 200 OK
18:12:36.122013 IP 10.244.1.9.80 > 10.244.2.173.56304: Flags [P.], seq 239:854, ack 75, win 487, options [nop,nop,TS val 340410407 ecr 501951332], length 615: HTTP
...
```


```
# worker-2
$ tcpdump -i ens5 -n udp port 8472 and host 10.13.1.57 -vv -c 20
18:12:36.122205 IP (tos 0x0, ttl 64, id 19884, offset 0, flags [none], proto UDP (17), length 102)
10.13.1.57.42553 > 10.13.1.129.8472: [no cksum] OTV, flags [I] (0x08), overlay 0, instance 26328
IP (tos 0x0, ttl 64, id 34355, offset 0, flags [DF], proto TCP (6), length 52)
10.244.2.173.56304 > 10.244.1.9.80: Flags [F.], cksum 0x5216 (correct), seq 75, ack 854, win 483, options [nop,nop,TS val 501951333 ecr 340410407], length 0
18:12:36.122252 IP (tos 0x0, ttl 64, id 56492, offset 0, flags [none], proto UDP (17), length 102)
10.13.1.129.53433 > 10.13.1.57.8472: [no cksum] OTV, flags [I] (0x08), overlay 0, instance 9764
IP (tos 0x0, ttl 64, id 20241, offset 0, flags [DF], proto TCP (6), length 52)
10.244.1.9.80 > 10.244.2.173.56304: Flags [F.], cksum 0x5211 (correct), seq 854, ack 76, win 487, options [nop,nop,TS val 340410407 ecr 501951333], length 0
...
```


另外還記得我們在 Day11 提到的工具箱嗎？

使用 cilium-dbg 的監控工具來監控封包怎麼跑的，以我們這裡想追封包的路徑不想要看到太多雜訊，所以會加上 `--type=trace`

option

找到 worker-1 和 worker-2 各自的 Cilium Pod 然後連線進去

```
# 在兩個 Node 各自的 cilium-agent 裡執行
$ cilium-dbg monitor --type=trace
```


接著發出請求：

```
# Client Pod 向 Server Pod 發出請求
kubectl exec client -- curl -s http://10.244.1.9
```


實際的 trace 輸出如下，由於輸出內容比較多，我只挑其中幾個來展示：

```
# worker-1 cilium-agent (發送端)
-> overlay flow 0xae1c1868 , identity 26328->9764 state new ifindex cilium_vxlan orig-ip 0.0.0.0: 10.244.2.173:41600 -> 10.244.1.9:80 tcp SYN
-> endpoint 1617 flow 0x67ea74a6 , identity 9764->26328 state reply ifindex lxc3a6f7d4e8c73 orig-ip 10.244.1.9: 10.244.1.9:80 -> 10.244.2.173:41600 tcp SYN, ACK
# worker-2 cilium-agent (接收端)
-> endpoint 2805 flow 0xbcd7b6e , identity 26328->9764 state new ifindex lxcaa66993d2495 orig-ip 10.244.2.173: 10.244.2.173:42058 -> 10.244.1.9:80 tcp SYN
-> overlay flow 0xa2c5733c , identity 9764->26328 state reply ifindex cilium_vxlan orig-ip 0.0.0.0: 10.244.1.9:80 -> 10.244.2.173:42058 tcp SYN, ACK
```


先說明一下上方看到的 Identity 對應的是哪個 Pod (可以用 `kubectl get cep`

去查 Identity)：

- client Pod: Identity 26328
- server Pod: Identity 9764

再來我們來解析不同視角

**worker-1 cilium-agent (發送端) 的視角**：

-
`overlay flow ... ifindex cilium_vxlan ... SYN`

→ 代表 client Pod (10.244.2.173) 發的封包在 worker-1 被 BPF 處理後

**封裝進 VXLAN**，準備丟到對方節點 -
`endpoint ... ifindex lxc3a6f7d4e8c73 ... SYN, ACK`

→ 這是 server Pod 回覆的 SYN/ACK，被 worker-1 收到 VXLAN 封包、解封裝後，最後打到 client Pod veth 介面

✅ 所以在發送端，你先看到「封裝」，再看到「對方回應的解封裝」


**worker-2 cilium-agent (接收端) 的視角**：

-
`endpoint ... ifindex lxcaa66993d2495 ... SYN`

→ client Pod 的封包解開 VXLAN 後，最後打到 server Pod 的 veth (這裡是 lxcaa66993d2495)，所以這裡會記錄

`endpoint flow`

-
`overlay flow ... ifindex cilium_vxlan ... SYN, ACK`

→ server Pod 回應的封包 (SYN/ACK) 被 BPF 處理後，準備封裝成 VXLAN 再丟回 worker-1，所以記錄成

`overlay flow`

✅ 所以在接收端，你先看到「解封裝」，再看到「自己 Pod 的回應被封裝」


前面好幾篇 Cilium Pod to Pod Datapath 系列文，我們花了很多篇幅在講理論、trace 原始碼、拆解每一段 Datapath 的細節。這篇文章算是 Pod-to-Pod Datapath 系列的收尾 —— 不只停留在原始碼的分析，而是透過實際驗證，把理論對照到真實環境裡的封包流動。

而透過原始碼分析和實際驗證，我們也可以確定一件事：**跨 Node Pod-to-Pod 的流量，其實完全不會經過 cilium_host，而是直接從 Pod veth 被 redirect 到 cilium_vxlan 做封裝**。


這代表什麼？其實就是 eBPF 的「程式化 Datapath」威力完全展現。Cilium 把這段邏輯下放到 Kernel，透過 BPF 程式在不同 Hook 點接手，直接決定封包的命運，做到低延遲、低 overhead 的跨 Node 通訊。

**關於今天文章這裡的幾個核心 takeaway：**

-
是跨 Node 路由決策的核心程式碼`handle_ipv4_from_lxc()`

-
**VXLAN 封裝參數**並不是靜態設定，而是由 BPF 動態決定 -
**BPF 程式分工明確**，不同流量類型（Pod、Host、Overlay）都有專屬處理邏輯 -
**效能最佳化設計**：直接`redirect`

減少多餘 hop，最大化 throughput

理解這些細節，對後續要玩 Cilium 的效能調優、debug 網路異常，甚至研究更進階的功能都會非常關鍵。
