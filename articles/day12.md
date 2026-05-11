# [DAY12] 探索 Cilium Pod to Pod Datapath (1) 背後竟然有 ARP 偽造？

> 原文連結：https://ithelp.ithome.com.tw/articles/10387997

---

DAY
12

Cloud Native
### 30 天深入淺出 Cilium ：從入門到實戰系列 第
12 篇

##
[Day 12] 探索 Cilium Pod to Pod Datapath (1) 背後竟然有 ARP 偽造？


接下來我們就要來探索 Datapath 了，在探索之前，有一些話想跟讀者說：

- Cilium 是以 eBPF 技術為底實現出來的 CNI Plugin，而 eBPF 造就的「可程式化 Datapath」會使得我們必須「
**實際去看原始碼**」才可以知道封包是怎麼被 BPF Program 左右命運的 - 你所使用的 Cilium 可能有配置不同的參數，或者你的 Linux Kernel 版本、sysctl 參數都會讓你看到的 Datapath 與我看到的不相同，只能說 Networking 的世界真的是相當複雜，但同時也是 Networking 的樂趣之一

如果你已經做好要看大量原始碼的心理準備！那就跟著我開始探索 **Cilium Pod-to-Pod Datapath**

`client Pod@worker-1`

→ `server Pod@worker-1`


```
$ kubectl get po -o wide
NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES
client 1/1 Running 21 (59m ago) 24h 10.244.2.173 worker-1 <none> <none>
server 1/1 Running 0 24h 10.244.2.61 worker-1 <none> <none>
root@master-1:~#
```


讓我們試著以 client Pod 的思維去看這世界，請先假裝自己是 client pod

連進去 client Pod:

```
kubectl exec -it client -- sh
```


```
# client pod
$ ip route
default via 10.244.2.161 dev eth0 mtu 8951
10.244.2.161 dev eth0 scope link
```


-
**重點 1**：client pod (10.244.2.173) 的**default gateway**是`10.244.2.161`

→ 在這個 /32 或 /24 Pod 網段裡，Cilium 幫 Pod 安排一個「gateway IP」

-
**重點 2**：所有不是自己 IP 的封包（包含送往 server: 10.244.2.61）都會先丟給這個 gateway (10.244.2.161)。

也就是說，client Pod **不會直接找 server Pod** ，而是交給 **10.244.2.161**。

```
# client pod
$ ip neigh
10.244.2.161 dev eth0 lladdr 92:e2:d7:db:d2:0e STALE
```


- 這行表示：
- 下一跳 (gateway) 的 IP =
`10.244.2.161`

- 對應的 MAC =
`92:e2:d7:db:d2:0e`


- 下一跳 (gateway) 的 IP =

我們剛剛確認了：

-
**client Pod**(10.244.2.173) → route 的 default gateway =`10.244.2.161`

- ARP 表示這個 gateway IP 對應的 MAC =
`92:e2:d7:db:d2:0e`


也就是說，**client Pod 發往 server Pod (10.244.2.61) 的封包，第一站一定會送到 10.244.2.161**

待會我們來找一下 `10.244.2.161`

究竟是誰

如何找到 Pod 的 host-side veth 我們 Day 11 的工具箱已經有教學了，以下就不贅述，在 worker-1 上執行以下指令：

```
# worker-1
# 找 client pod 的 container id
# sudo crictl ps | grep client
# 用 container id 找出 PID
# sudo crictl inspect <container-id> | grep -i pid
# 進去 container netns，列出介面
$ sudo nsenter -t 293362 -n ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
25: eth0@if26: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default
link/ether ca:16:6c:38:b8:ca brd ff:ff:ff:ff:ff:ff link-netnsid 0
```


-
**Pod 內**：`eth0`

(ifindex 25)，MAC =`ca:16:6c:38:b8:ca`

-
**@if26**：表示這個`eth0`

的 peer 在 Node 上的介面 ifindex = 26

也就是說：

- client Pod 的 eth0 對端，在 Node (worker-1) 上是 ifindex 26 的 veth。
- 這就是 client Pod 發出去的封包，第一個會撞到的地方。

然後順便找看看 10.244.2.161 是誰：

```
$ ip addr show | grep -A1 -B2 10.244.2.161
5: cilium_host@cilium_net: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default qlen 1000
link/ether 76:3e:c7:69:39:9a brd ff:ff:ff:ff:ff:ff
inet 10.244.2.161/32 scope global cilium_host
valid_lft forever preferred_lft forever
```


也順便看看 `92:e2:d7:db:d2:0e`

是誰：

```
$ ip link | grep -B1 92:e2:d7:db:d2:0e
26: lxc3a6f7d4e8c73@if25: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
link/ether 92:e2:d7:db:d2:0e brd ff:ff:ff:ff:ff:ff link-netns cni-b64fd411-7115-e309-3395-20697029e00e
```


一樣我們在 worker-1 上，執行以下指令：

```
# worker-1
# 找出 ifindex=26 的網卡
$ ip link | grep -A1 '^26:'
26: lxc3a6f7d4e8c73@if25: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
link/ether 92:e2:d7:db:d2:0e brd ff:ff:ff:ff:ff:ff link-netns cni-b64fd411-7115-e309-3395-20697029e00e
```


接著用 bpftool 來看網卡上綁了哪些 BPF Program:

```
$ bpftool net
xdp:
tc:
ens5(2) tcx/ingress cil_from_netdev prog_id 4600 link_id 18
ens5(2) tcx/egress cil_to_netdev prog_id 4602 link_id 19
cilium_net(4) tcx/ingress cil_to_host prog_id 4583 link_id 17
cilium_host(5) tcx/ingress cil_to_host prog_id 4577 link_id 15
cilium_host(5) tcx/egress cil_from_host prog_id 4576 link_id 16
cilium_vxlan(6) tcx/ingress cil_from_overlay prog_id 4513 link_id 13
cilium_vxlan(6) tcx/egress cil_to_overlay prog_id 4514 link_id 14
lxc_health(8) tcx/ingress cil_from_container prog_id 4565 link_id 20
lxc3a6f7d4e8c73(26) tcx/ingress cil_from_container prog_id 4773 link_id 29
lxc72ce9b67c76c(28) tcx/ingress cil_from_container prog_id 4776 link_id 30
flow_dissector:
netfilter:
```


其中裡面就可以看到 `lxc3a6f7d4e8c73`

(這就是剛剛對應 client Pod eth0@if26 的對端)：

-
**hook**：`tcx/ingress`

-
**程式**：`cil_from_container`


也就是說：client Pod 發出去的封包，第一站就是 hit 這個 `cil_from_container`

程式

關於 cil_from_container 是放什麼？

**現在我們可以拼出第一段 Datapath：**

- client Pod eth0 丟封包 → gateway (10.244.2.161)
- 到達 Node veth 對端
`lxc3a6f7d4e8c73(26)`

- 在
**ingress**立刻被 Cilium 掛的 BPF prog接住`cil_from_container`

- 這裡做的事：
- 讀取封包 metadata → identity → policy 判斷
- 如果目標是同 Node 的 server Pod → 查
`endpoint_map`

→ 找到對應 veth - 然後 local deliver
- 如果目標是跨 Node → 會轉交
`cilium_vxlan`

/`ens5`

path


- 這裡做的事：

先在 Worker-1 執行以下指令， `lxc3a6f7d4e8c73`

是 client Pod 的 host-side veth:

```
# worker-1
sudo tcpdump -n -i lxc3a6f7d4e8c73 arp or icmp -vv -c 30
```


接著在 client pod 去 ping server pod

```
# client pod
ping -c 3 10.244.2.61
```


我們會在 tcpdump 指令下看到：

```
# worker-1
$ sudo tcpdump -n -i lxc3a6f7d4e8c73 arp or icmp -vv -c 30
tcpdump: listening on lxc3a6f7d4e8c73, link-type EN10MB (Ethernet), snapshot length 262144 bytes
14:48:12.086356 IP (tos 0x0, ttl 64, id 21258, offset 0, flags [DF], proto ICMP (1), length 84)
10.244.2.173 > 10.244.2.61: ICMP echo request, id 39, seq 1, length 64
14:48:13.094882 IP (tos 0x0, ttl 64, id 21721, offset 0, flags [DF], proto ICMP (1), length 84)
10.244.2.173 > 10.244.2.61: ICMP echo request, id 39, seq 2, length 64
14:48:14.119887 IP (tos 0x0, ttl 64, id 22053, offset 0, flags [DF], proto ICMP (1), length 84)
10.244.2.173 > 10.244.2.61: ICMP echo request, id 39, seq 3, length 64
14:48:17.126859 ARP, Ethernet (len 6), IPv4 (len 4), Request who-has 10.244.2.161 tell 10.244.2.173, length 28
14:48:17.126884 ARP, Ethernet (len 6), IPv4 (len 4), Reply 10.244.2.161 is-at 92:e2:d7:db:d2:0e, length 28
```


這裡有一點神奇的事情：

-
`cilium_host`

interface 前面觀察是真的有綁`10.244.2.161/32`

- 但 ARP 回覆的 MAC 並不是 cilium_host (
`76:3e:c7:69:39:9a`

)，而是**client Pod 對端 veth (lxc3a6f7d4e8c73)**的 MAC (`92:e2:d7:db:d2:0e`

)

那這說明了什麼？

-
**10.244.2.161 的邏輯身份**= Pod 的「gateway IP」，掛在`cilium_host`

上，這是 route 層面的概念。 -
**10.244.2.161 的實際行為**= 當 Pod 發 ARP request，並不是`cilium_host`

介面（kernel）回應，而是**veth 對端 ingress 的 Cilium BPF 程式**

為什麼會這樣？？**其實是我們前面看到綁在 lxc3a6f7d4e8c73 身上的 BPF Program cil_from_container 在搞鬼**。原始碼放在 /bpf/bpf_lxc.c，就是我們在 Day 10 講到的


`bpf_lxc.c`

。`cil_from_container`

`cil_from_container`

在做什麼？簡單一句話來解釋就是：「**cil_from_container 是容器網路的出口警衛，專門檢查所有從容器送出的封包，並決定它們的下一步該怎麼走**。」當一個容器送出一個封包時，這個 BPF 程式會被第一個觸發，而裡面有用到 **eBPF Tail call** 的機制根據不同封包類型來轉給不同的 BPF Program 接力處理。

對於 eBPF tail call 細節感興趣的讀者可以參考此文件。簡單說 Tail call 就像是 BPF Program 的「接力賽」，讓一個 BPF 程式可以把封包 (接力棒) 交給下一個 BPF Program 繼續處理


以下是 Code Snippet:

```
// bpf/bpf_lxc.c
__section_entry
int cil_from_container(struct __ctx_buff *ctx)
{
__u16 proto = 0; // 這裡用來暫存 L3 protocol
int ret;
// 先確認封包的 EtherType，順便把結果存到 proto（像 ETH_P_IP 這種）
validate_ethertype(ctx, &proto);
// 再來依照不同 L3 protocol 分流
switch (proto) {
#ifdef ENABLE_IPV4
// 如果是 IPv4 封包就會進來這裡
// 例如 tcpdump 看到 "proto ICMP"，因為 ICMP 算是 IP 的一部分
case bpf_htons(ETH_P_IP):
// 把封包丟給專門處理 IPv4 policy 的 BPF 程式做進一步檢查
ret = tail_call_internal(ctx, CILIUM_CALL_IPV4_FROM_LXC, &ext_err);
break;
// 如果是 ARP 封包就會進到這裡
#ifdef ENABLE_ARP_PASSTHROUGH
case bpf_htons(ETH_P_ARP):
// pass-through 模式：ARP 直接放行
ret = CTX_ACT_OK;
break;
#elif defined(ENABLE_ARP_RESPONDER)
case bpf_htons(ETH_P_ARP):
// responder 模式：交給專門的 ARP BPF 程式處理
ret = tail_call_internal(ctx, CILIUM_CALL_ARP, &ext_err);
break;
#endif // ENABLE_ARP_RESPONDER
#endif // ENABLE_IPV4
default:
// 其他不支援的 L3 protocol 全部丟掉
ret = DROP_UNKNOWN_L3;
}
return ret;
}
```


接著看一下 tail_handle_arp，一樣在 bpf/bpf_lxc.c，原始碼不會很長，我們可以直接看，我把英文註解部分轉成中文：

```
// bpf/bpf_lxc.c
__declare_tail(CILIUM_CALL_ARP)
int tail_handle_arp(struct __ctx_buff *ctx)
{
union macaddr mac = THIS_INTERFACE_MAC; // veth 對端的 MAC 位址
union macaddr smac;
__be32 sip; // Sender IP
__be32 tip; // Target IP
/* 若 ARP 封包不合法，交回 Linux stack 處理 */
if (!arp_validate(ctx, &mac, &smac, &sip, &tip))
return CTX_ACT_OK;
/*
* 預期 Pod 會對它的 gateway IP 發 ARP Request
* 大部分情況下是 IPV4_GATEWAY (例如 10.244.2.161)
* 若 target IP 剛好是 Pod 自己的 IP，則不要回覆
* （避免誤判成 IP 重複檢查）
*/
if (tip == CONFIG(endpoint_ipv4).be32)
return CTX_ACT_OK;
/* 偽造一個 ARP Reply：
* 回覆「tip 這個 IP 的擁有者，就是 THIS_INTERFACE_MAC」
*/
return arp_respond(ctx, &mac, tip, &smac, sip, 0);
}
```


到這裡為止，稍微總結一下：

- client Pod 發 ARP request →
`cil_from_container`

接住 -
`cil_from_container`

看到是`ETH_P_ARP`

→ tail call 到`tail_handle_arp()`

-
`tail_handle_arp()`

不透過 Linux proxy_arp，直接呼叫`arp_respond()`

，用 veth 的 MAC 假裝自己是 gateway

所以 Pod 看到的 gateway 10.244.2.161，其實從頭到尾只是 **BPF 在騙它**，把「路由的 IP 概念」綁到「veth 的 MAC」

這樣會可以提高效能，因為 BPF 在 TC Ingress 就能直接回 ARP 封包，省掉從 netdev → kernel network stack → 再回來的多次 context switch

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
