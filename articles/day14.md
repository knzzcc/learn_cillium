# [DAY14] 探索 Cilium Pod to Pod Datapath (3) cilium_host, cilium_net, cilium_vxlan 是什麼？

> 原文連結：https://ithelp.ithome.com.tw/articles/10389328

---

DAY
14

Cloud Native
### 30 天深入淺出 Cilium ：從入門到實戰系列 第
14 篇

##
[Day 14] 探索 Cilium Pod to Pod Datapath (3) cilium_host, cilium_net, cilium_vxlan 是什麼？


前幾天我們主要在看「同 Node 的 Pod-to-Pod Datapath」。這算是最基本、最單純的情境，但光這個情境就能明顯感受到 eBPF 帶來的 **可程式化 datapath**。

不過在真實環境裡，很多流量其實是「跨 Node 通信」，這就比單 Node 要複雜得多了。讀者們在前面探索 Pod-to-Pod Datapath 時，有沒有注意到 `ip link show`

的輸出裡會冒出一些 `cilium_host`

、`cilium_net`

、`cilium_vxlan`

的介面？

這些帶著 `cilium`

前綴的 netdev，我之前都沒細講，因為「同 Node 的 Pod-to-Pod Datapath」其實用不到它們。但一旦進到「跨 Node 通信」的情境，這些介面就會派上用場。

所以在進入跨 Node Datapath 之前，這篇文章先來帶大家認識一下 `cilium_host`

、`cilium_net`

、`cilium_vxlan`

各自扮演的角色。

在 Cilium 的 datapath 實現中，有三個關鍵的網路介面負責處理不同層面的網路流量：

-
和`cilium_host`

: 一對 veth pair，連接 Cilium IP 空間與主機網路模型`cilium_net`

`# worker-1 $ ip link | grep -A1 cilium 4: cilium_net@cilium_host: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default link/ether 02:8d:02:06:64:af brd ff:ff:ff:ff:ff:ff 5: cilium_host@cilium_net: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP mode DEFAULT group default qlen 1000 link/ether 76:3e:c7:69:39:9a brd ff:ff:ff:ff:ff:ff # ...`

-
: VXLAN 隧道介面，用於跨節點的 overlay 網路通信`cilium_vxlan`


| 特性/介面 | cilium_host |
cilium_net |
cilium_vxlan |
|---|---|---|---|
介面類型 |
veth | veth | VXLAN |
主要用途 |
連接 cilium IP space 與 host networking | veth pair 的另一端，處理特定流量 | 跨 Node overlay 通信 |
IP 地址 |
✅ 有 (Pod 內部 default gateway IP) | ❌ 無 | ❌ 無 |
BPF 程式 |
`cil_to_host` (ingress) / `cil_from_host` (egress) |
`cil_to_host` (ingress) |
overlay 封裝/解封裝 |
流量類型 |
Host ↔ Pod / L7 Proxy 流量 / NodePort/LB 服務 | 特定的 host firewall 和網路設備相關流量 | Pod-to-Pod (跨 Node) / VXLAN 封裝流量 |


補充說明：host = Node 本身的 root network namespace，也就是這台機器的 Linux OS 自己的 Network Stack

這些介面構成了 Cilium 網路架構的核心基礎設施，每個都承擔著特定的角色和功能。

根據 `pkg/defaults/node.go`

中的定義：

原始碼連結點我


```
// HostDevice is the name of the device that connects the cilium IP
// space with the host's networking model
HostDevice = "cilium_host"
// SecondHostDevice is the name of the second interface of the host veth pair.
SecondHostDevice = "cilium_net"
```


這對 veth pair 是 Cilium datapath 的核心組件，用於：

-
連接 Cilium 管理的 IP Space 與 host networking model

-
提供 BPF 程式的掛載點，實現流量處理和 Policy 執行

-
所以使用

`bpftool net`

指令可以觀察到這對 veth pair 的 TC ingress (或 egress) 都有被綁 BPF 程式：`# worker-1 $ bpftool net tc: # 略 ... cilium_net(4) tcx/ingress cil_to_host prog_id 4583 link_id 17 cilium_host(5) tcx/ingress cil_to_host prog_id 4577 link_id 15 cilium_host(5) tcx/egress cil_from_host prog_id 4576 link_id 16 # 略 ...`

-
像是上面的指令輸出就看到分別有兩個 BPF 程式，後面實際探索跨 Node Datapath 時我們會細節查看兩個 BPF 程式的原始碼：

-
`cil_to_host`

: 處理離開主機的流量，執行 Egress Policy 和路由決策 -
`cil_from_host`

: 處理進入主機的流量，執行 Ingress Policy 和 Proxy redirect

-

-
-
High Level 的處理流程大概是這樣：

`# cil_from_host: 處理「從 cilium_host 出去」的流量 (TC egress) 某處 -> cilium_host -> cil_from_host (TC egress) -> 下一跳 # cil_to_host: 處理「進入 cilium_host/cilium_net」的流量 (TC ingress) 某處 -> cilium_host/cilium_net -> cil_to_host (TC ingress) -> 下一跳`

-
作為路由和 Policy 決策的關鍵節點


在 `pkg/datapath/loader/netlink.go`

中的 `setupBaseDevice()`

函式負責創建這對 veth pair：

原始碼連結點我


```
// pkg/datapath/loader/netlink.go
// setupBaseDevice decides which and what kind of interfaces should be set up as
// the first step of datapath initialization, then performs the setup (and
// creation, if needed) of those interfaces. It returns two links and an error.
// By default, it sets up the veth pair - cilium_host and cilium_net.
func setupBaseDevice(logger *slog.Logger, sysctl sysctl.Sysctl, mtu int) (netlink.Link, netlink.Link, error) {
if err := setupVethPair(logger, sysctl, defaults.HostDevice, defaults.SecondHostDevice); err != nil {
return nil, nil, err
}
linkHost, err := safenetlink.LinkByName(defaults.HostDevice)
if err != nil {
return nil, nil, err
}
linkNet, err := safenetlink.LinkByName(defaults.SecondHostDevice)
if err != nil {
return nil, nil, err
}
// 關閉 ARP
// 不會回應 ARP Request（外界問「誰是 10.244.2.161？」→ kernel 不會自動回）
// 也不會主動對其他 IP 發 ARP Request
// 詳細原因可以回去複習 Day 12 文章
if err := netlink.LinkSetARPOff(linkHost); err != nil {
return nil, nil, err
}
if err := netlink.LinkSetARPOff(linkNet); err != nil {
return nil, nil, err
}
// 設定 MTU
if err := netlink.LinkSetMTU(linkHost, mtu); err != nil {
return nil, nil, err
}
if err := netlink.LinkSetMTU(linkNet, mtu); err != nil {
return nil, nil, err
}
return linkHost, linkNet, nil
}
```


`cilium_host`

是 Cilium 路由器的主要介面，具有以下特性：

-
`cilium_host`

會被分到 IP address，也就是在 Pod Network Namespace 裡面查看 routing table 時，default route 的 IP address：原始碼連結點我

`// pkg/datapath/loader/netlink.go // addHostDeviceAddr add internal ipv4 and ipv6 addresses to the cilium_host device. func addHostDeviceAddr(hostDev netlink.Link, ipv4, ipv6 net.IP) error { if ipv4 != nil { addr := netlink.Addr{ IPNet: &net.IPNet{ IP: ipv4, Mask: net.CIDRMask(32, 32), // corresponds to /32 }, } if err := netlink.AddrReplace(hostDev, &addr); err != nil { return err } } // IPv6 處理類似... }`

-
**BPF 程式掛載**: 在`pkg/datapath/loader/loader.go`

中，`cilium_host`

身上掛載了關鍵的 BPF 程式：原始碼連結點我

`// pkg/datapath/loader/loader.go // attachCiliumHost inserts the host endpoint's policy program into the global // cilium_call_policy map and attaches programs from bpf_host.c to cilium_host. func attachCiliumHost(logger *slog.Logger, ep datapath.Endpoint, lnc *datapath.LocalNodeConfiguration, spec *ebpf.CollectionSpec) error { // ... 程式載入邏輯 ... // Attach cil_to_host to cilium_host ingress. if err := attachSKBProgram(logger, host, hostObj.ToHost, symbolToHostEp, bpffsDeviceLinksDir(bpf.CiliumPath(), host), netlink.HANDLE_MIN_INGRESS, option.Config.EnableTCX); err != nil { return fmt.Errorf("interface %s ingress: %w", ep.InterfaceName(), err) } // Attach cil_from_host to cilium_host egress. if err := attachSKBProgram(logger, host, hostObj.FromHost, symbolFromHostEp, bpffsDeviceLinksDir(bpf.CiliumPath(), host), netlink.HANDLE_MIN_EGRESS, option.Config.EnableTCX); err != nil { return fmt.Errorf("interface %s egress: %w", ep.InterfaceName(), err) } }`


`cilium_net`

是 veth pair 的另一端，主要用於：

-
**作為到 Host 的 redirect 目標**: 當流量目標是`HOST_ID`

時，會被重定向到`CILIUM_NET_IFINDEX`

的 ingress (`ctx_redirect(ctx, CILIUM_NET_IFINDEX, BPF_F_INGRESS)`

) -
**BPF 程式掛載**: 在**TC ingress**掛載`cil_to_host`

程式，使用`NetdevMapName`

（與 cilium_host 的`HostMapName`

不同） -
**與 cilium_host 協同工作**: cilium_host egress 處理後可能將封包推向 cilium_net ingress 進行進一步處理

原始碼連結點我


```
// pkg/datapath/loader/loader.go
// attachCiliumNet attaches programs from bpf_host.c to cilium_net.
func attachCiliumNet(logger *slog.Logger, ep datapath.Endpoint, lnc *datapath.LocalNodeConfiguration, spec *ebpf.CollectionSpec) error {
// ... 取得 cilium_net 介面 ...
// Attach cil_to_host to cilium_net.
if err := attachSKBProgram(logger, net, netObj.ToHost, symbolToHostEp,
bpffsDeviceLinksDir(bpf.CiliumPath(), net), netlink.HANDLE_MIN_INGRESS, option.Config.EnableTCX); err != nil {
return fmt.Errorf("interface %s ingress: %w", defaults.SecondHostDevice, err)
}
}
```


```
// pkg/defaults/node.go
// VxlanDevice is a device of type 'vxlan', created by the agent.
VxlanDevice = "cilium_vxlan"
```


`cilium_vxlan`

是一個 VXLAN 類型的網路介面，用於實現跨節點的 overlay 網路功能。

在 `pkg/datapath/loader/netlink.go`

中的 `setupVxlanDevice()`

函式負責創建 VXLAN 介面：

原始碼連結點我


```
// pkg/datapath/loader/netlink.go
// setupVxlanDevice ensures the cilium_vxlan device is created with the given
// port, source port range, and MTU.
func setupVxlanDevice(logger *slog.Logger, sysctl sysctl.Sysctl, port, srcPortLow, srcPortHigh uint16, mtu int) error {
mac, err := mac.GenerateRandMAC()
if err != nil {
return err
}
dev := &netlink.Vxlan{
LinkAttrs: netlink.LinkAttrs{
Name: defaults.VxlanDevice,
MTU: mtu,
HardwareAddr: net.HardwareAddr(mac),
},
FlowBased: true,
Port: int(port),
PortLow: int(srcPortLow),
PortHigh: int(srcPortHigh),
}
// 檢查是否需要重新創建（例如 port 變更）
if l, err := safenetlink.LinkByName(dev.Attrs().Name); err == nil {
vxlan, _ := l.(*netlink.Vxlan)
if vxlan.Port != int(port) {
if err := netlink.LinkDel(l); err != nil {
return fmt.Errorf("deleting outdated vxlan device: %w", err)
}
}
}
l, err := ensureDevice(logger, sysctl, dev)
if err != nil {
return fmt.Errorf("creating vxlan device %s: %w", dev.Attrs().Name, err)
}
}
```


- Cilium 把 VXLAN 隧道控制權交給 BPF，每個封包的封裝/解封裝都可以動態決定，靠
`ctx_set_tunnel_key()`

/`ctx_get_tunnel_key()`

操作（這些函式最終會調用內核的`bpf_skb_set_tunnel_key()`

/`bpf_skb_get_tunnel_key()`

helper）

| 特性 | 傳統 VXLAN |
Cilium 的 VXLAN |
|---|---|---|
配置方式 |
預先配置所有 VTEP | 動態決定 tunnel 參數 |
FDB |
需要靜態 FDB (Forwarding Database) entry | 不需要預先配置 |
封裝控制 |
固定的封裝參數 | BPF 程式動態控制 |
適用場景 |
靜態網路環境 | 動態容器環境 |

-
**動態配置**: 支援動態改變 destination port，但會重新創建介面 -
**Source Port Range**: 支援設定來源 port 範圍，但僅在首次創建時有效

`cilium_vxlan`

會在以下場景中發揮作用：

-
**跨 Node 通信**: 用於封裝 pod-to-pod 的跨 Node 流量 -
**故障排除**: 可以使用`tcpdump -n -i cilium_vxlan`

來監控跨 Node 流量 -
**加密支援**: 在啟用 IPSec 時，VXLAN 流量會進行加密處理

以下列出幾個比較典型而且會跟今天主題 `cilium_host`

, `cilium_net`

, `cilium_vxlan`

有關的路徑，**但是實際路徑真的會根據 CIlium 的參數配置或是網路環境有所差異**，不代表任何人安裝的 Cilium 都是這樣走

-
**Pod 到 Host**:- Pod 流量 →
`lxcXXX`

→`cilium_host`

→ Host Network Stack

- Pod 流量 →
-
**Host 到 Pod**:- Host network stack →
`cilium_host`

→ Pod

- Host network stack →
-
**Pod 到 Pod (跨 Node)**:- Pod 流量 →
`lxcXXX`

→`cilium_vxlan`

→ 物理介面 → 對端 Node

- Pod 流量 →

基於 `bpf_host.c`

原始碼的實際功能：

-
**cilium_host ingress**:`cil_to_host`

- 處理 L7 Proxy 重定向、IPSec 處理、Host Firewall ingress policy 等 -
**cilium_host egress**:`cil_from_host`

- 處理來自 host namespace 的流量，執行跨節點路由決策、Host Firewall egress policy、L7 Proxy 標記處理等 -
**cilium_net ingress**:`cil_to_host`

- 同樣的 BPF 程式，但使用不同的 map（NetdevMapName vs HostMapName）

從 `daemon/cmd/daemon_main.go`

中可以看到相關的配置選項：

```
// EnableEndpointRoutes 預設是 false
flags.Bool(option.EnableEndpointRoutes, defaults.EnableEndpointRoutes,
"Use per endpoint routes instead of routing via cilium_host")
flags.Int(option.RouteMetric, 0,
"Overwrite the metric used by cilium when adding routes to its 'cilium_host' device")
```


Cilium 會自動為這些介面設定適當的路由：

```
// worker-1
// EnableEndpointRoutes = false ，實際看到的 route 範例
// 注意看 Cilium 相關的 route 沒有寫 metric，意思是它們的 metric 預設是 0
$ ip route
default via 10.13.1.1 dev ens5 proto dhcp src 10.13.1.57 metric 100
10.13.0.2 via 10.13.1.1 dev ens5 proto dhcp src 10.13.1.57 metric 100
10.13.1.0/24 dev ens5 proto kernel scope link src 10.13.1.57 metric 100
10.13.1.1 dev ens5 proto dhcp scope link src 10.13.1.57 metric 100
10.244.0.0/24 via 10.244.2.161 dev cilium_host proto kernel src 10.244.2.161 mtu 8951
10.244.1.0/24 via 10.244.2.161 dev cilium_host proto kernel src 10.244.2.161 mtu 8951
10.244.2.0/24 via 10.244.2.161 dev cilium_host proto kernel src 10.244.2.161
10.244.2.161 dev cilium_host proto kernel scope link
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
```


```
// 呼叫 Map 的命名規則
"cilium_calls": bpf.LocalMapName(callsmap.HostMapName, uint16(ep.GetID())) // cilium_host
"cilium_calls": bpf.LocalMapName(callsmap.NetdevMapName, uint16(ifindex)) // cilium_net 和外部介面
```


如果有一些參數的 default 值不知道是什麼，可以去 `pkg/defaults/defaults.go`

找，像是我想要找 `EnableEndpointRoutes`

參數 default 值就可以從這裡找到是 `false`


這三個 netdev 基本上就是 Cilium datapath 的核心 infra：

-
**cilium_host / cilium_net**：Cilium 和 Host network 之間的橋樑，既是 Pod netns 裡的 Gateway IP，也是 Policy enforcement 的關鍵位置。 -
**cilium_vxlan**：提供跨 Node 的 overlay network，讓不同 Node 上的 Pod 可以彼此互通。

理解這些介面的角色和實作細節，是掌握 Cilium 網路架構、做 troubleshooting 的必備知識。

而且因為 Cilium 倚賴 eBPF 帶來的「**可程式化 datapath**」，要真的弄懂這些魔法，光看文件不夠，還得 dive into 原始碼才能體會背後的設計。

今天把這三個 netdev 條理清楚之後，接下來我們就可以開始探索 **跨 Node 的 Datapath**。
