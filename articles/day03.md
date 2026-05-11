# [DAY03] 認識 Cilium 核心組件

> 原文連結：https://ithelp.ithome.com.tw/articles/10380950

---

DAY
3

Day 2 我們已經建立一個 K8s Cluster，也透過 Helm 安裝好 Cilium 了，那你可能會想知道「安裝了哪些東西」，那我們現在就可以搭配實際操作認識一下 Cilium 的核心組件

今天我們會先基本認識這些組件，更深入的細節與底層原理我們將在更後面的文章說明～

請先起兩個 Pods

```
apiVersion: v1
kind: Pod
metadata:
  name: client
  labels:
    app: client
spec:
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
  containers:
	  - name: server
	    image: nginx
```


然後請連線到任一個 worker node

## Cilium CNI (cilium-cni)

這就是跟著 CNI 實作出來的 CNI Plugin，所以就是一個 Binary 檔案，每個 Node 身上都會有這個 Binary 檔案，我們可以在 /opt/cni/bin/ 目錄下找到它

```
$ ls -l /opt/cni/bin/cilium-cni

-rwxr-xr-x 1 root root 69714464 Sep 14 19:01 /opt/cni/bin/cilium-cni
```


所以當今天起一個 Pod 之後，Container runtime 會執行 CNI Plugin，被執行的 CNI Plugin 就是 Cilium CNI (cilium-cni)

Cilium CNI 會配置好基本的網路 (例如：建立 veth pair、將 Pod 網卡移動到 Pod 的 Network Namespace…)，至於大部分的網路設定其實都是 Cilium Agent 來配置的，例如：Assign Pod IP

而 Cilium CNI 要和 Cilium Agent 溝通，背後其實是透過 REST API 向 Cilium Agent 發出請求，像是要建立 Endpoint 的話，Cilium CNI 會對 Cilium Agent 發出 `PUT /endpoint/{id}`

請求。

原始碼：https://github.com/cilium/cilium/blob/v1.18.1/plugins/cilium-cni/cmd/cmd.go

是 Cilium 中，非常重要的組件，他會以 DaemonSet 的形式存在在每個 Node 上，負責的事情非常多，包括：

- Assign Pod IP
- 建立 Endpoint
- 建立 CIliumNode
- 分配 Identity
- 管理 BPF Program
- 管理 BPF Maps
- …

要做的事情其實非常多，且實作細節也比較複雜，所以在 Day 3 我認為只要大概知道 Cilium Agent 主要負責：

- 管理 BPF Prog / Maps
- Datapath 設定
- Custom Resource 生命週期管理

細節我們等後面有合適的情境才來細講！

我們可以透過以下指令來找到 Cilium Agent

```
kubectl get ds cilium -n kube-system
```


然後可以透過以下指令看一下 cilium-agent 做了什麼，以下面來說，我是趁剛建立 Pod 時，擷取一小段 Logs ：

```
$ kubectl -n kube-system logs <cilium-agent-pod> --tail=50
# ... 略 ...
time=2025-09-16T15:16:09.221863342Z level=info msg="Create endpoint request" module=agent.controlplane.endpoint-api addressing="&{IPV4:10.244.1.119 IPV4ExpirationUUID:0dd8387c-67bd-471a-90dd-d2c5d2e7760f IPV4PoolName:default IPV6: IPV6ExpirationUUID: IPV6PoolName:}" containerID=21bb37c412396104f779527a646203c350845bad8d15de647a649ee997f408ce containerInterface=eth0 datapathConfiguration="&{DisableSipVerification:false ExternalIpam:false InstallEndpointRoute:false RequireArpPassthrough:false RequireEgressProg:false RequireRouting:<nil>}" interface=lxc0c1189139188 k8sPodName=default/server k8sUID=127bf793-da88-4132-ae63-94ffaeafc8f2 labels=[] sync-build=true
time=2025-09-16T15:16:09.222973251Z level=info msg="New endpoint" desiredPolicyRevision=0 ipv6="" ipv4=10.244.1.119 endpointID=681 ciliumEndpointName=default/server k8sPodName=default/server containerInterface="" containerID=21bb37c412 datapathPolicyRevision=0 subsys=endpoint
# ... 略 ...
```


原始碼：https://github.com/cilium/cilium/tree/v1.18.1/daemon

Cilium Operator 我覺得可以先理解成後勤支援的角色，會以 Deployment 的形式存在在 K8s Cluster，可以有多個 Replicas，但是只有一個會作為 Leader，例如：當我們使用 EKS 搭配 AWS ENI IPAM Mode，Cilium Operator 就會負責和 AWS 交互 (例如：創建 ENI、AssignPrivateIPAddress 等)，然後把 AWS 那邊拿到的 IP 放到一個 Custom Resource (CR) 叫做 CiliumNode。而 Cilium Agent 要 Assign IP 給 Pod 就會去看 CiliumNode 那邊有哪些可用 IP，這些可用 IP 就是 Cilium Operator 幫我們跟 AWS 拿的

我們可以透過以下指令來找到 Cilium Operator:

```
kubectl get deploy cilium-operator -n kube-system
```


Hubble 的主要職責是為 Cilium 提供網路可觀測性 (Network Observability)。

有這個真的很方便，例如：想要刪掉某個服務時，有了 Hubble 可以方便觀察這個 Pod 還有誰正在打，或是在導入 Network Policy 時，透過 Hubble 即時觀察流量也可以觀察到 Network Policy 有沒有生效

Hubble 由以下幾個部分組成

-
**Hubble Server**：內嵌於 Cilium Agent 中，直接從 eBPF 收集網路流量數據，並提供 gRPC API 供查詢。
**Hubble Relay**：一個獨立的組件，它會自動發現 Cluster 中所有的 Hubble Server，並提供一個單一的、Cluster 範圍的 API 端點
**Hubble UI**：一個強大的 Web 圖形介面，可以動態生成服務依賴關係圖，即時顯示服務間的網路流量、HTTP 請求、DNS 查詢，甚至是 Network Policy 的阻擋情況。 
![[Pasted image 20260511164244.png]]
**Hubble CLI**：一個命令列工具，讓開發者和維運人員可以方便地從終端機中查詢和過濾即時的網路流量事件。

可以用以下指令找到 hubble-relay, hubble-ui

```
kubectl get deploy -n kube-system | grep hubble
```

## 小結
---
**Cilium CNI**: 和 Contianer runtime 交互，處理最基本網路配置，會透過 REST API 和 Cilium Agent 溝通 -
**Cilium Agent**: 在每個 Node 上執行具體的網路與安全任務 -
**Cilium Operator**: 後勤支援、管理和協調 IP -
**Hubble**: 提供網路可觀測性 (Network Observability)

