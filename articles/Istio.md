
**Control Plane — istiod**

就是一個單一的程式（以前是拆成 Pilot、Citadel、Galley 三個元件，後來合併成 istiod）。它做三件事：把你寫的 VirtualService 等 YAML 轉換成 Envoy 能懂的設定（xDS API）推送下去、負責發放和輪換 mTLS 憑證、還有處理服務發現（知道哪些 Pod 在哪裡）。

**Data Plane — Envoy Proxy**

真正幹活的。每個 Pod 被注入一個 Envoy sidecar container，所有進出 Pod 的流量都被它攔截。路由、負載均衡、mTLS 加解密、重試、熔斷、收集 metrics，全是它在做。

兩者的關係很像 Cilium 的架構：Cilium 的 operator 相當於 control plane，每個 node 上的 cilium-agent 相當於 data plane。差別在於 Cilium 是 per-node 一個 agent，Istio 傳統模式是 per-pod 一個 Envoy。

Ambient mode 則是把 data plane 再拆成兩層：ztunnel（per-node，處理 L4）加上 waypoint proxy（按需部署，處理 L7），這樣就不用每個 Pod 都塞一個 Envoy 了。

CNI負責 L3/L4
	Pod 拿 IP，Pod間能不能通，Network Policy IP/port層級的隔離
Service Mesh 管L7
	HTTP路由，header-based決策，mTLS身分，可觀測性(非侵入式)


但Cillium野心很大
CilliumNetworkPolicy 可以過濾HTTP path/metod，Hubble可以做L7觀測性，Cillium Service Mesh L7路由

Istio ambient mode 往下做 CNI 處理的部分，ztunnel做per-node 的L4處理

基本的L7過濾和可觀測性，Cillium可以滿足
按比例切流量做金絲雀，複雜的retry/timeout鏈，fault injection測試，跨級群的服務治理
Istio的VirtualService/DestinationRule功能比較強大



---
**第一階段：理解 Istio 的架構與核心概念**

先搞清楚 Istio 和 Cilium 的定位差異。Cilium 偏重 L3/L4 層、以 eBPF 為核心；Istio 則是以 Envoy sidecar（或 ambient mode 的 ztunnel）處理 L7 流量。建議從官方文件的架構頁開始，理解 istiod（control plane）、Envoy proxy（data plane）、以及 ambient mesh 這三個核心元件的關係。

**第二階段：動手裝起來**

用 `istioctl` 在本地的 kind 或 minikube 叢集裝一套。先用 demo profile 就好，搭配官方的 Bookinfo 範例應用跑一遍。這一步的重點是感受 sidecar injection 的運作方式，觀察 `istio-proxy` container 怎麼被注入到每個 Pod。

**第三階段：流量管理（Istio 最強的部分）**

這是 Istio 和 Cilium 差異最大的地方，也是 Istio 的核心價值。依序學這些 CRD：

VirtualService 和 DestinationRule 是最重要的兩個，掌握 request routing、traffic shifting（金絲雀部署）、fault injection、timeout/retry。Gateway 則負責南北向流量（ingress）。建議每個都實際寫 YAML 跑一次，不要只看文件。

**第四階段：可觀測性**

裝上 Kiali、Jaeger、Prometheus、Grafana 這套觀測 stack。因為你學過 Cilium 的 Hubble，可以對比兩者在可觀測性上的差異。Kiali 的服務拓撲圖很直覺，這是 Istio 生態的一大優勢。

**第五階段：安全性（mTLS 與授權）**

了解 PeerAuthentication（mTLS 模式）和 AuthorizationPolicy。Istio 預設就會幫你做 mTLS，但你需要理解 STRICT vs PERMISSIVE 模式的差異，以及怎麼寫細粒度的授權策略。這部分可以和 Cilium 的 NetworkPolicy 做對比。

**第六階段：Ambient Mesh（新趨勢）**

Istio 的 ambient mode 是目前社群的重點方向，用 ztunnel（per-node）取代 sidecar，L7 處理用 waypoint proxy。這個模式在概念上其實更接近 Cilium 的做法（per-node agent），學起來你會特別有感覺。




---


[超越 Gateway API：深入探索 Envoy Gateway 的扩展功能](https://www.zhaohuabing.com/post/2024-08-31-introducing-envoy-gateways-gateway-api-extensions/)
https://www.zhaohuabing.com/archive/
ambient
https://jimmysong.io/zh/blog/istio-ambient-terminology/
[超越 Sidecar：深入解析 Istio Ambient Mode 的流量机制与成本效益](https://jimmysong.io/zh/blog/beyond-sidecar/)

[深入解析 Istio Ambient 模式中的流量路径：eBPF 和 Istio 的强力结合](https://jimmysong.io/zh/blog/istio-ambient-ebpf/)
深入探讨 Kubernetes 集群中 Cilium、eBPF 与 Istio Ambient 模式的完美结合。本文详细解析在无 kube-proxy 环境下的流量路径，提供完整的部署指南和最佳实践。

istio.io/latest/docs/ambient/

[An In-Depth Look at Istio Ambient Mode with Calico](https://www.tigera.io/blog/an-in-depth-look-at-istio-ambient-mode-with-calico/)

当K8S容器没有bash时，7种高阶排查手段（生产环境亲测有效）

[BGP in Kubernetes with Cilium Why and How - Rastislav Szabo](https://www.youtube.com/watch?v=2KXSiwCpC7Q)

Docker Swarm沒有namespace?
差Ingress/GateWay API整近來
常見的k8s工具
Trobuleshooting
gitops argocd




Ingress 最初的設計目標很單純——處理簡單的 HTTP 路由，所以 API 規範非常精簡。但隨著實際需求越來越複雜（流量拆分、header 改寫、gRPC、TCP 路由等），這些功能都無法塞進原有的 API 結構裡。結果就是各家 Controller 廠商各自用 annotation 來擴展，導致同一個需求在不同 Controller 下的寫法完全不同，形成了嚴重的碎片化。

要在 Ingress API 上做根本性的改進，等於要打破向後相容性，而 Kubernetes 對 GA API 的穩定性承諾非常嚴格，不可能大改。所以社群選擇了「另起爐灶」，也就是 Gateway API。


### Ingress Controller 的角色

回到上一個話題，Ingress 資源本身只是一份**描述文件**，它說「請把 `example.com/api` 的流量導到 backend-service」。但 Kubernetes 本身不會幫你做這件事，你需要一個 **Ingress Controller** 來實際執行。

Ingress Controller 做的事情就是：監聽叢集裡所有的 Ingress 資源，然後根據裡面的規則去配置一個反向代理（通常是 Nginx、Envoy、HAProxy 等）。所以流量的實際處理者是那個反向代理，而 Controller 負責把你寫的 Ingress 規則「翻譯」成反向代理能懂的配置。

常見的 Ingress Controller 包括 ingress-nginx（社群版）、Traefik、HAProxy Ingress、Kong 等，你至少要部署一個，Ingress 資源才會生效。


### NodePort

這是 Kubernetes Service 的一種類型。它會在叢集中每個 Node 上開放一個固定的端口（範圍 30000-32767），外部流量打到任一 Node 的那個端口，就能到達對應的 Service。

用途很直接但比較粗糙。通常用在開發測試，或是自己前面有一層外部 Load Balancer 的情況。缺點是端口範圍有限、沒有 L7 路由能力、也沒有 TLS 終止。

### LoadBalancer

也是 Service 的一種類型。建立後，雲端供應商會自動幫你建一個外部的負載均衡器（例如 GCP 的 Network Load Balancer），分配一個外部 IP，流量從這個 IP 進來再分發到後端 Pod。

比 NodePort 方便很多，但每建一個 LoadBalancer 類型的 Service，就會產生一個獨立的外部負載均衡器和 IP。如果你有 20 個服務，就會有 20 個負載均衡器，成本很高。而且它通常是 L4 層級的，不做基於路徑或主機名的路由。

### Ingress

解決的就是上面 LoadBalancer 的問題。Ingress 是一個 L7 路由層，讓你可以用**一個**入口點，根據 host 和 path 把流量分發到不同的 Service。比如 `example.com/api` 到 Service A，`example.com/web` 到 Service B，共用同一個外部 IP 和負載均衡器。

它需要一個 Ingress Controller 來運作（上一則聊過的）。


### 當你建立一個 LoadBalancer 類型的 Service 時

你 apply 一個 YAML，Kubernetes 就會去呼叫雲端供應商的 API。以 GKE 為例，流程大致是這樣：

**第一步**，Kubernetes 裡有一個叫 Cloud Controller Manager 的元件，它是 GKE 幫你裝好的，專門負責跟 GCP API 溝通。它偵測到你建了一個 LoadBalancer 類型的 Service，就會去呼叫 GCP 的 API。

**第二步**，GCP 收到請求後，真的在它的基礎設施上建立一個 Network Load Balancer，分配一個外部公網 IP（例如 `35.201.x.x`）。這個負載均衡器是跑在 Google 的網路基礎設施上的，完全在 Kubernetes 叢集外面。

**第三步**，這個負載均衡器的後端目標（backend）會被設定指向你叢集中各個 Node 的 NodePort。所以本質上 LoadBalancer 類型的 Service 底層還是會開一個 NodePort，只是雲端多幫你做了一層——在前面放了一個真正的負載均衡器來接流量。

**流量路徑就是**：使用者打 `35.201.x.x` → GCP 的 Load Balancer → 選一個 Node 的 NodePort 轉進去 → kube-proxy 再把流量導到正確的 Pod。



KIND練習argocd / istio / cillium / calico 會有不足的地方嗎?
怎麼拆專案，具體步驟


KIND練習argocd / istio / cillium / calico 會有不足的地方嗎?

Claude responded: 會，KIND 能跑起來這些東西，但每個都有程度不等的限制。我分別講。

會，KIND 能跑起來這些東西，但每個都有程度不等的限制。我分別講。

### ArgoCD
這個在 KIND 上跑起來**最沒問題**。ArgoCD 本質上就是一個跑在叢集裡的應用，監聽 Git repo 然後做部署，不太依賴底層網路或硬體。你練習 GitOps 流程、App of Apps 模式、sync policy 這些都完全沒問題。唯一小差異是 KIND 沒有真正的 LoadBalancer，所以存取 ArgoCD UI 要用 port-forward 或 NodePort，但這不影響學習。

### Istio
能裝能跑，但有幾個地方會卡。首先 Istio 的 IngressGateway 預設是 LoadBalancer 類型的 Service，在 KIND 上拿不到外部 IP，要自己改成 NodePort 或用 MetalLB。其次 Istio 的 sidecar 注入會讓每個 Pod 多跑一個 Envoy proxy，記憶體開銷不小，KIND 跑在你的筆電上，節點多一點就容易吃光資源。再來，Istio 的一些進階功能像 multi-cluster mesh、外部服務的 egress gateway 整合，在 KIND 上很難模擬真實場景。不過學 VirtualService、DestinationRule、流量拆分、mTLS 這些核心概念是夠用的。

### Cilium
這個限制比較明顯。Cilium 依賴 eBPF，而 eBPF 是跑在 Linux kernel 上的。KIND 的節點是 Docker 容器，它們共用宿主機的 kernel。所以實際上 eBPF 程式是載入到你宿主機的 kernel 裡，不是載入到一個獨立的 Node kernel。如果你的宿主機是 macOS 或 Windows，Docker Desktop 底下其實跑了一個 Linux VM，kernel 版本不一定夠新，某些 eBPF 功能可能不支援或行為不一致。

Cilium 的一些進階功能像 bandwidth manager、BBR 擁塞控制、host firewall、XDP 加速，在 KIND 上基本跑不了或跑了也沒意義，因為沒有真正的網路硬體。BGP peering、LoadBalancer IP 分配這些跟外部網路整合的功能也無法練習。但基本的 NetworkPolicy、L7 policy、Hubble 觀測性是可以玩的。

### Calico
跟 Cilium 類似的問題但程度輕一些，因為 Calico 預設用 iptables 模式，不像 Cilium 那麼依賴 eBPF。在 KIND 上裝 Calico 做 NetworkPolicy 練習是完全可行的。限制主要在 BGP 模式——Calico 可以跟外部路由器做 BGP peering 來通告 Pod 網段，這在 KIND 上沒辦法練。另外 Calico 的 eBPF dataplane 模式跟 Cilium 會遇到一樣的問題。

### 整體建議
KIND 適合學習這些工具的核心概念和基本操作，對考 CKA/CKS 或理解架構夠用。但如果你要練的是接近生產環境的場景，像多節點高可用、雲端整合、真實的網路效能調校，那用雲端的免費額度（GKE free tier、EKS 等）或者用 multipass / Vagrant 開幾台真正的 VM 跑 kubeadm 會更貼近實際。資源不夠的話，單台 Linux 機器跑 k3s 配 Cilium 也是一個折衷的好選擇。


[监控改造之路4](https://capops.xyz/observe4)

[Kubernetes CKA 與CKS 認證考試心得分享 - Blueswen](https://blueswen.com/blog/2026/02/20/kubernetes-cka-%E8%88%87-cks-%E8%AA%8D%E8%AD%89%E8%80%83%E8%A9%A6%E5%BF%83%E5%BE%97%E5%88%86%E4%BA%AB/)
CKA 395 美金 + CKS 395 美金 = **790 美金**，換算台幣大約 **25,000 左右**（看匯率）。

推vmware

24g 283g空閒，2600
Hyper-V - 韌體中已啟用虛擬化 否
Claude responded: 那就是 BIOS 裡的虛擬化沒開，開一下就好。
空間夠，就是要額外安裝，BIOS要開，CPU也大概夠用

deskmini 硬碟線 $300

缺sata線 12400，power電源線直出

azure30天 gcp90天甚麼機型都可以開
aws 12個月，但是有限機型，開不了k8s

感覺考k8s 5張證照比刷leetcode簡單多了，leetcode現場考不一定是原題，用抽的

kind argocd在12400玩
其他的上gke開一下服務，不一定集中在一個月
地端 vm  腳本
	rufus os推薦
	安裝Linux CX2 240g on 2600 缺SATA線
	安裝vmware
m73弄cluster但是都要硬碟 $+++

硬客高手  sata 35
ubunto desktop
ubuntu 22.04LTS

這個問題核心就是網路架構怎麼設定，讓 Windows host 能連到 Linux 裡 VMware 的 VM 以及 VM 上跑的 K8s 服務。

有幾層要處理：

**1. VMware 網路模式選 Bridged**

最簡單的做法。VM 會直接拿到跟你 Windows 同網段的 IP，Windows 直接就能連。三台 VM 都設 Bridged，它們之間也能互通。

如果你的網路環境不方便用 Bridged（比如公司網路有 MAC 管控），就用 **NAT + port forwarding** 或 **Host-Only + Linux 上做路由**。

**2. Host-Only + 路由方案（更常見的做法）**

在 VMware 建一個 Host-Only 網段，例如 `192.168.100.0/24`，三台 VM 都掛這個網段。然後在 Linux 上確認 IP forwarding 開著：

bash

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

接著在 Windows 上加一條靜態路由，把 VM 網段指向 Linux 的 IP：

cmd

```cmd
route add 192.168.100.0 mask 255.255.255.0 <Linux的IP> -p
```

這樣 Windows 就能直接連到所有 VM 了。

**3. K8s Service 的存取**

K8s 的 Service 預設是 ClusterIP，只有 cluster 內部能連。要從外面連進去：

- **NodePort** — Service 開在每台 node 的某個 port（30000-32767），Windows 直接連 `VM的IP:NodePort` 就行。
- **MetalLB** — 如果你想玩 LoadBalancer type，裝 MetalLB 分配一段 IP pool（從你 Host-Only 網段切一段出來），這樣 Service 會拿到獨立 IP，最接近真實環境。

**4. 簡化存取（可選）**

在 Windows 的 `C:\Windows\System32\drivers\etc\hosts` 加上 VM 的 hostname 對應，就不用每次記 IP：

```
192.168.100.10  k8s-master
192.168.100.11  k8s-worker1
192.168.100.12  k8s-worker2
```

**總結最推薦的組合：**

Host-Only 網段 + Windows 加靜態路由 + K8s 用 NodePort 或 MetalLB。這樣最乾淨，不依賴外部網路，隨時都能連。

通常 Ubuntu Desktop 裝完 SSH server 後預設就允許密碼登入，但如果不行就改一下設定：
```bash
sudo apt install openssh-server
sudo systemctl enable --now ssh
```

確認 `/etc/ssh/sshd_config` 裡面這行：
```
PasswordAuthentication yes
```

如果被設成 `no` 或被註解掉，改成 `yes` 然後：
```bash
sudo systemctl restart ssh
```


SSH
注音 --> 裝成倉頡?
```bash
sudo apt install ibus-chewing language-pack-zh-hant language-pack-gnome-zh-hant
```

VMWare網路

遠端連線
sudo apt install xrdp
sudo apt install tigervnc-standalone-server -y


Master：  4 CPU / 8GB RAM / 15GB disk
Worker1： 2 CPU / 6GB RAM / 15GB disk
Worker2： 2 CPU / 6GB RAM / 15GB disk


vb terraform支援度比 vmware好，但是社群維護的有時候還是會怪怪
vSphere/ESXi
- ESXi 是 bare-metal hypervisor，裝下去整台機器就只能當 VM host，你的 Ubuntu 桌面沒了
- 你要額外一台電腦或筆電透過 vSphere Client 網頁去管理
- 免費版 ESXi 功能被砍很多，Broadcom 收購後授權更混亂


init snapshot
```
# 1. 更新系統（最花時間，看網速）
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl openssh-server net-tools

# 2. 關 swap
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

# 3. 載入 kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# 4. sysctl 設定
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system

# 5. 裝 containerd
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd

# 6. 裝 kubeadm kubelet kubectl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# 7. 禁用 cloud-init 網路
echo "network: {config: disabled}" | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

改hostname

```bash
# 改 hostname
sudo hostnamectl set-hostname master

# 確認
hostnamectl
```

三台分別改：
```bash
# master
sudo hostnamectl set-hostname master

# worker1
sudo hostnamectl set-hostname worker1

# worker2
sudo hostnamectl set-hostname worker2
```

改網路
```bash
sudo nano /etc/netplan/01-static.yaml
```

**master：**
```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.137.101/24
      routes:
        - to: default
          via: 192.168.137.1
      nameservers:
        addresses:
          - 8.8.8.8
```

設定vm時間utc8 taipei
```bash
sudo timedatectl set-timezone Asia/Taipei
```

確認：
```bash
timedatectl
```

免密碼 在windows上
ssh-keygen -t ed25519
```
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh user@192.168.1.100 "cat >> .ssh/authorized_keys"
```

1433 或長的