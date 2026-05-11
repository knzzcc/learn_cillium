# [DAY11] 探索 Cilium Datapath 前，你需要的工具箱

> 原文連結：https://ithelp.ithome.com.tw/articles/10387354

---

DAY
11

前面幾天我們已經知道當建立 Pod 時，Cilium 在背後做了什麼。

那網路都配置好之後，究竟一個封包從 Pod A 發送給 Pod B 經歷了什麼呢？

接下來會準備 Datapath 系列，但是在探索 Datapath 前，我希望先來熟悉一些好用的工具，以避免在講解 Cilium Datapath 時，我會需要塞很大的篇幅講解這個指令是在做什麼的。一方面也可以驗證我們先前的小小系列文「當一個 Pod 被建立時 Cilium 在背後做了什麼？」在熟悉工具的同時順便實際觀察這些 Cilium 幫我們做的東西

我這裡建立了兩個 Pods 並確保都在 worker-1：

- client Pod (netshoot)
- server Pod (nginx)

```
apiVersion: v1
kind: Pod
metadata:
name: client
labels:
app: client
spec:
nodeName: worker-1 # 指定 Schedule 到 woerker-1
containers:
- name: client
image: nicolaka/netshoot
command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
name: server
labels:
app: server
spec:
nodeName: worker-1 # 指定 Schedule 到 woerker-1
containers:
- name: server
image: nginx
```


接著再另外開一個 Terminal 並連線進去 worker-1

找出 Client 網卡:

```
# worker-1
sudo crictl ps
```


拿到 Container ID 後，找出 PID

```
# worker-1
sudo crictl inspect <CONTAINER_ID> | grep -i pid
```


利用 `nsenter`

找到 Pod 對端的 veth (host-side veth)

```
# worker-1
# 以我自己為例我找到 client pod 的 PID 是 265553
sudo nsenter -t <PID> -n ethtool -S eth0 | grep peer_ifindex
```


從上面的指令會看到 ifindex，你就可以知道 client Pod 的 host-side veth 的 index

```
# worker-1
ip link | grep <peer_ifindex>
```


以我這裡為例 peer_ifindex 是 26：

```
# worker-1
$ ip link | grep 26
26: lxc3a6f7d4e8c73@if25: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000
```


```
bpftool version
```


沒安裝的話請安裝

```
sudo apt update
sudo apt install linux-tools-$(uname -r) -y
```


觀察一下有綁定哪些 BPF Program

```
# worker-1
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
lxc7d9779c20d88(16) tcx/ingress cil_from_container prog_id 4553 link_id 24
lxc037abb0f6833(18) tcx/ingress cil_from_container prog_id 4547 link_id 25
lxc3a6f7d4e8c73(26) tcx/ingress cil_from_container prog_id 4773 link_id 29 # 會注意到 26 在 tcx 上
lxc72ce9b67c76c(28) tcx/ingress cil_from_container prog_id 4776 link_id 30
flow_dissector:
netfilter:
```


也可以來看一下有哪些 maps

```
# worker-1
$ bpftool map show
24: perf_event_array name cilium_events flags 0x0
key 4B value 4B max_entries 2 memlock 280B
pids cilium-agent(10347)
25: hash name cilium_ratelimi flags 0x1
key 4B value 8B max_entries 64 memlock 1984B
pids cilium-agent(10347)
26: hash name cilium_node_map flags 0x1
key 20B value 4B max_entries 16384 memlock 263632B
pids cilium-agent(10347)
27: prog_array name cilium_call_pol flags 0x0
key 4B value 4B max_entries 65535 memlock 524544B
owner_prog_type sched_cls owner jited
28: prog_array name cilium_egressca flags 0x0
key 4B value 4B max_entries 65535 memlock 524544B
owner_prog_type sched_cls owner jited
# ... 略 ...
```


傳統是使用 tc，但是現在新版本的 Cilium 會檢查如果 Kernel 夠新就會使用 tcx，不夠新 fallback 使用 tc。

關於 tc 與 tcx 差異，我先用比喻的方式解釋 tc：

- Kernel 的 ingress/egress hook point 就是插座
- tc 就是一個多功能轉接頭，這轉接頭上面有孔給你掛 BPF Prog
- tc 會插在插座上，BPF Prog 插在 tc 上

接下來 tcx 就好理解了，就是不透過 tc 間接插上去 ingress/egress hook point，而是可以「**直接**」插上去。

可以執行以下指令，會發現沒有任何 output

```
# worker-1
# 用剛剛 client 的 host side veth 網卡
tc filter show dev lxc3a6f7d4e8c73 ingress
```


連線進去 worker-1 的 Cilium Pod

先來看 IP 對應到 Identity 的 BPF Map `ipcache`


```
# cilium-agent pod
$ cilium-dbg bpf ipcache list | grep 10.244.2.173 # client pod IP
10.244.2.173/32 identity=26328 encryptkey=0 tunnelendpoint=0.0.0.0 flags=<none>
```


```
Available Commands:
auth Manage authenticated connections between identities
bandwidth BPF datapath bandwidth settings
config Manage runtime config
ct Connection tracking tables
egress Manage the egress routing rules
endpoint Local endpoint map
frag Manage the IPv4 datagram fragments
fs BPF filesystem mount
ipcache Manage the IPCache mappings for IP/CIDR <-> Identity
ipmasq ip-masq-agent CIDRs
lb Load-balancing configuration
metrics BPF datapath traffic metrics
multicast Manage multicast BPF programs
nat NAT mapping tables
nodeid Manage the node IDs
policy Manage policy related BPF maps
recorder PCAP recorder
sha Manage compiled BPF template objects
socknat Socket NAT operations
srv6 Manage the SRv6 routing rules
vtep Manage the VTEP mappings for IP/CIDR <-> VTEP MAC/IP
```


然後我們在這篇文章前面其實有教學怎麼在 Worker Node 上透過 `crictl`

一路用到 nsenter 來找到 host-side veth 的 index，但是其實在 Cilium Agent Pod 裡面可以直接查看 `endpoint`

BPF Map

```
# cilium-agent pod
$ cilium-dbg bpf endpoint list
IP ADDRESS LOCAL ENDPOINT INFO
10.244.2.161:0 (localhost)
10.244.2.173:0 id=1617 sec_id=26328 flags=0x0000 ifindex=26 mac=CA:16:6C:38:B8:CA nodemac=92:E2:D7:DB:D2:0E parent_ifindex=0 # 可注意 ifindex，是 26
10.244.2.20:0 id=258 sec_id=12196 flags=0x0000 ifindex=32 mac=76:B4:62:BA:AD:D5 nodemac=DE:F2:3F:BD:76:75 parent_ifindex=0
10.244.2.234:0 id=180 sec_id=4325 flags=0x0000 ifindex=30 mac=96:F8:72:97:3D:79 nodemac=BE:01:CC:5C:38:72 parent_ifindex=0
10.244.2.99:0 id=3262 sec_id=4 flags=0x0000 ifindex=8 mac=4A:57:D9:4D:31:5B nodemac=0A:3D:7C:82:BD:B9 parent_ifindex=0
10.244.2.61:0 id=133 sec_id=9764 flags=0x0000 ifindex=28 mac=02:AE:3E:DE:75:41 nodemac=7E:F0:89:A2:90:42 parent_ifindex=0
10.13.1.57:0 (localhost)
```


再來是有一個很讚的是 `metrics`

:

```
# cilium-agent pod
$ cilium-dbg bpf metrics list
REASON DIRECTION PACKETS BYTES LINE FILE
Interface INGRESS 12540314 2034158833 1156 bpf_host.c
Interface INGRESS 331452 24703853 753 bpf_overlay.c
Interface INGRESS 37899 1591758 1075 bpf_host.c
LB: sock cgroup: Reverse entry delete succeeded EGRESS 64 0 1256 bpf_sock.c
Policy denied EGRESS 134 9916 1361 bpf_lxc.c
Policy denied INGRESS 47 3550 2127 bpf_lxc.c
Policy denied by denylist EGRESS 24 1776 1361 bpf_lxc.c
Policy denied by denylist INGRESS 5 394 2127 bpf_lxc.c
Success EGRESS 1281 151362 1305 bpf_lxc.c
Success EGRESS 153145 11287570 1808 bpf_host.c
Success EGRESS 2532035 276466543 459 nodeport_egress.h
Success EGRESS 330 77656 122 local_delivery.h
Success EGRESS 331471 24771063 35 encap.h
Success EGRESS 370493 43066536 1648 bpf_host.c
Success EGRESS 7371245 2670131232 1344 bpf_lxc.c
Success INGRESS 10050402 860540645 2030 bpf_lxc.c
Success INGRESS 9872160 847536155 122 local_delivery.h
Unknown L4 protocol EGRESS 47 9491 449 nodeport_egress.h
Unsupported L3 protocol EGRESS 1013 71802 1544 bpf_lxc.c
```


GitHub Repo: https://github.com/cilium/pwru

安裝方式可以自己編譯，或是直接下載 binary

這是自己編譯的安裝方式：

```
# worker-1
sudo apt update
sudo apt install -y clang llvm gcc make flex bison byacc yacc libpcap-dev golang
git clone https://github.com/cilium/pwru.git
cd pwru
make
```


或直接下載 binary，注意底下依照你的 Node 可能要把 amd64 改成 arm64

```
# worker-1
curl -L -O https://github.com/cilium/pwru/releases/download/v1.0.10/pwru-linux-amd64.tar.gz
```


```
tar -xzvf pwru-linux-amd64.tar.gz
sudo mv pwru /usr/local/bin/
pwru --version
```


下載好後，簡單展示一下用法

以我們剛才有找到 client Pod 的 PID 是 265553 簡單示範一下 `--filter-netns`

選項：

```
# 撈特定 network namespace
pwru --filter-netns "/proc/265553/ns/net"
```


也簡單展示一下 `--filter-ifname`

選項：

```
# 撈指定 interface
# 可搭配前面學的 --filter-netns 一起使用
# 預設使用當前 netns
$ pwru --filter-ifname "lxc3a6f7d4e8c73"
SKB CPU PROCESS NETNS MARK/x IFACE PROTO MTU LEN TUPLE FUNC
0xffff91ba905e1ce8 1 ~bin/curl:272299 4026531840 0 ~3a6f7d4e8c73:26 0x0800 9001 60 10.244.2.173:54864->10.244.2.61:80(tcp) __netif_rx
```


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
