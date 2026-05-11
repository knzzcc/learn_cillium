# [DAY02] Cilium 安裝與初體驗

> 原文連結：https://ithelp.ithome.com.tw/articles/10380611

---

DAY
2

Day 1 已經認識了 Cilium 要解決的痛點和問題，Day 2 我們將來在 K8s Cluster 安裝 Cilium，為了不綁定在特定雲端廠商的 K8s Distribution，所以我會選擇使用 VM + kubeadm 的方式來建立 K8s Cluster。

- 3 台 Ubuntu 24 VM (1 control plane + 2 worker nodes)，可以自由選擇任意方式（例如：AWS EC2, 自己的電腦開 VM）

-
Arc: x86_64

-
Linux Distribution: Ubuntu 24.04.3 LTS

-
Linux Kernel: 6.14.0-1011-aws

-
K8s

`Client Version: v1.32.9 Kustomize Version: v5.5.0 Server Version: v1.32.9`

-
containerd

`[containerd.io](http://containerd.io/) 1.7.27 05044ec0a9a75232cad458027ca83437aae3f4da`

-
kubeadm

`kubeadm version: &version.Info{Major:"1", Minor:"32", GitVersion:"v1.32.9", GitCommit:"cea7087b31eb788b75934d769a28f058ab309318", GitTreeState:"clean", BuildDate:"2025-09-09T19:44:20Z", GoVersion:"go1.23.12", Compiler:"gc", Platform:"linux/amd64"}`


首先我們先連線進去其中 1 台 VM，這台 VM 要作為 control plane

接著依照 [已確定可運作] 在 AWS EC2 自己建立 Single node Cluster 安裝即可

Control Plane 配置好之後，會出現以下指令，記得保留好後面會用到：

```
kubeadm join 10.13.1.243:6443 --token 2j6u0d.x6nd6vwl3h5mwmt3 \
--discovery-token-ca-cert-hash sha256:6ef26555a8ba65424a603b18ab34b9068d5d838f2e9971b254934a74d1bcb701
```


如果找不到上面的指令，那請執行以下指令：

```
kubeadm token create --print-join-command
```


請參考：Worker Node 如何 Join Cluster

如果都沒問題，照理來說在 Control Plane 會看到以下輸出：

```
$ kubectl get no
NAME STATUS ROLES AGE VERSION
control-plane NotReady control-plane 10m v1.32.9
worker-1 NotReady <none> 8m6s v1.32.9
worker-2 NotReady <none> 3m46s v1.32.9
```


連線進去 Control Plane

-
先確保安裝好 Helm，如果沒安裝 Helm 請參考官方文件，可以透過以下指令檢查

`helm version`

-
新增 Cilium Helm Repo

`helm repo add cilium https://helm.cilium.io/ helm repo update`

-
安裝 Cilium (我們把想玩的功能都開滿) ，記得下面有

`<control-plane-ip>`

要實際換成你的 IP address，例如:`--set k8sServiceHost=10.13.1.243`

，然後 clusterPoolIPv4PodCIDRList 不要跟 Node 網段一樣`helm install cilium cilium/cilium \ --namespace kube-system \ --version 1.18.1 \ --set kubeProxyReplacement=true \ --set k8sServiceHost=10.13.1.243 \ # <control-plane-ip> --set k8sServicePort=6443 \ --set ipam.mode=cluster-pool \ --set ipam.operator.clusterPoolIPv4PodCIDRList=10.244.0.0/16 \ --set ipam.operator.clusterPoolIPv4MaskSize=24 \ --set hubble.relay.enabled=true \ --set hubble.ui.enabled=true \ --set externalIPs.enabled=true \ --set nodePort.enabled=true \ --set hostPort.enabled=true \ --set bpf.masquerade=true \ --set enableIPv4Masquerade=true \ --set enableIPv6Masquerade=false \ --set policyAuditMode=false`

-
檢查 Cilium 各 Pods 是否 Running

`kubectl -n kube-system get po`


https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli

```
# 在 Control Plane
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```


https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#validate-the-installation

執行以下指令，確保所有狀態都 OK （沒看到任何紅字）

```
cilium status --wait
```


執行以下測試，確認所有測試通過，這測試會需要比較長的時間 (5 mins 左右)

```
cilium connectivity test
```


我們找出其中一個 Cilium Pod

```
kubectl -n kube-system get pods -l k8s-app=cilium
```


這裡我將創建兩個 Pods，一個是 `client`

(netshoot) 一個是 `server`

(NGINX)

我們透過指定 `.spec.nodeName`

來強行控制 Schedule 到哪個 Worker Node，這次我們先將 Pods 都調度到同一節點

建立 `test-pods.yaml`

:

```
vim test-pods.yaml
```


```
apiVersion: v1
kind: Pod
metadata:
name: client
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
spec:
nodeName: worker-1 # 指定 Schedule 到 woerker-1
containers:
- name: server
image: nginx
```


```
kubectl apply -f test-pods.yaml
```


檢查被調度到的節點

```
$ kubectl get po -o wide
NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES
client 1/1 Running 0 26s 10.244.2.136 worker-1 <none> <none>
server 1/1 Running 0 26s 10.244.2.239 worker-1 <none> <none>
```


接著來試試看從 `client`

Pod 向 `server`

Pod 發出 HTTP 請求是否能成功得到回應：

```
SERVER_IP=$(kubectl get po server -o jsonpath='{.status.podIP}')
kubectl exec client -- curl -s $SERVER_IP
```


現在來測試看看跨節點的 Pod-to-Pod 流量，我們把 Server 節點調度到 `worker-2`


先清除剛才的 Pods

```
kubectl delete -f test-pods.yaml
```


```
vim test-pods.yaml
```


```
apiVersion: v1
kind: Pod
metadata:
name: client
spec:
nodeName: worker-1
containers:
- name: client
image: nicolaka/netshoot
command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
name: server
spec:
nodeName: worker-2 # 指定 Schedule 到 woerker-2
containers:
- name: server
image: nginx
```


檢查是否被調度到不同 Worker Node:

```
$ kubectl get po -o wide
NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES
client 1/1 Running 0 4s 10.244.2.230 worker-1 <none> <none>
server 1/1 Running 0 4s 10.244.1.106 worker-2 <none> <none>
```


接著來試試看從 `client`

Pod 向 `server`

Pod 發出 HTTP 請求是否能成功得到回應：

```
SERVER_IP=$(kubectl get po server -o jsonpath='{.status.podIP}')
kubectl exec client -- curl -s $SERVER_IP
```


注意，不要在 Prodcution 這樣用

```
kubectl port-forward -n kube-system svc/hubble-ui --address 0.0.0.0 12000:80
```


```
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
HUBBLE_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then HUBBLE_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-${HUBBLE_ARCH}.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-${HUBBLE_ARCH}.tar.gz /usr/local/bin
rm hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
```


建議讀者們實際動手實操，可以看看安裝 Cilium 時有哪些東西被設定了，以下我給一些推薦觀察的路線，而我就不印出具體的 Output，這是為了鼓勵讀者真的動手去實際操作：

看看 Config Maps 多了什麼：

```
kubectl get cm -n kube-system
```


看看有哪些 CRD：

```
kubectl api-resources | grep cilium
```


觀察 ip rule

```
ip rule
```


觀察 ip route

```
ip route
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
