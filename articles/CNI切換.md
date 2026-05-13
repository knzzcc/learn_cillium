CNI 不能直接換，因為它管的是整個 Pod 網路，換了舊的路由規則、iptables、IPAM 全部會衝突。

最乾淨的做法：
1. worker 先 `kubeadm reset`
2. master 再 `kubeadm reset`
3. 清掉殘留的 CNI 設定
4. 重新 `kubeadm init`
5. 裝新的 CNI
6. worker 重新 join

整個過程大概 15-20 分鐘，因為 kubeadm 跟 containerd 都還在，不用重裝。

所以你之前拍的 snapshot 就很重要了。最理想的做法是在 `kubeadm init` 之前拍一個 snapshot，這樣要換 CNI 就直接還原到那個點，重新 init 選不同 CNI 就好，5 分鐘搞定。

建議的 snapshot 順序：
- **snap-1**：系統設好、kubeadm 裝好、還沒 init
- **snap-2**：Cilium 裝好
- **snap-3**：換 Calico 裝好
- **snap-4**：換 Flannel 裝好

這樣三個 CNI 都能隨時切換來比較


---
如果不想還原 snapshot，手動切換的流程：

**拆掉舊 CNI**

拆 Cilium：
1. cilium uninstall
2. 刪除 /etc/cni/net.d/ 底下的檔案
3. 刪除 /opt/cni/bin/ 底下 cilium 相關的 binary
4. 重啟每台 node 的 kubelet

拆 Calico：
1. kubectl delete -f calico.yaml 或刪掉 tigera operator
2. 刪除 /etc/cni/net.d/ 底下的檔案
3. 刪除 /opt/cni/bin/ 底下 calico 相關的 binary
4. 重啟每台 node 的 kubelet

拆 Flannel：
1. kubectl delete -f kube-flannel.yml
2. 刪除 /etc/cni/net.d/ 底下的檔案
3. 刪除 /opt/cni/bin/ 底下 flannel 相關的 binary
4. 每台 node 刪掉 flannel.1 介面：ip link delete flannel.1
5. 重啟 kubelet

**裝新 CNI**

Cilium：
1. cilium install
2. cilium status 確認

Calico：
1. kubectl create -f tigera-operator.yaml
2. apply calico custom resource
3. 等 pod 跑起來

Flannel：
1. kubectl apply -f kube-flannel.yml
2. 等 pod 跑起來

**注意事項**
pod-network-cidr 要對。kubeadm init 的時候指定的 CIDR 要跟 CNI 的設定一致。Calico 預設 192.168.0.0/16，Flannel 預設 10.244.0.0/16，Cilium 預設 10.0.0.0/8。如果 CIDR 不同，要在 CNI 設定裡改成你 init 時指定的那個。

**最推薦的做法**

不要手動拆，直接用 snapshot：
1. kubeadm init 完、還沒裝 CNI → 拍 snapshot「pre-cni」
2. 想換 CNI → 還原到 pre-cni → 直接裝新的

這樣最乾淨不會有殘留，省掉一堆 debug。所以等你 kubeadm init 完，記得先拍一個 snapshot 再裝 CNI。