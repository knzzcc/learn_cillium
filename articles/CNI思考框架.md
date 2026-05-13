玩CNI的思考框架，怎麼還原，怎麼安裝，怎麼驗證(包含各種層面的網路 有缺漏嗎?


框架很好，補一些細節：

**安裝前**
- 確認 kubeadm init 的 pod-network-cidr 跟你要裝的 CNI 相容
- 如果要換 CNI，先確認 cluster 是乾淨狀態還是已經有裝過別的 CNI
- 拍 snapshot

**安裝**
- 用官方推薦的方式裝（CLI 或 Helm）
- 確認 CNI 的 pod CIDR 跟 kubeadm init 時給的一致
- 等所有 CNI 相關 pod 都 Running

**驗證 — 你列了網路層面，展開來看：**
- Pod 層：同 node 的 Pod 之間能不能通
- Pod 跨節點：不同 node 的 Pod 之間能不能通
- Service 層：ClusterIP 解析跟連線正不正常
- DNS：Pod 裡面 nslookup kubernetes.default 有沒有回應
- 外部存取：NodePort 或 Ingress 從外面連得進來嗎
- Network Policy：設一條 deny all，確認流量真的被擋住
- CNI 專屬功能：Cilium 的話跑 cilium connectivity test，Calico 的話用 calicoctl node status

**還原 — 你有提到，補充具體步驟：**
- 最乾淨的方式：revert snapshot
- 手動還原：刪 CNI 資源（kubectl delete）→ 清每台 node 上的 CNI config（/etc/cni/net.d/）→ 清 iptables 殘留規則 → 重啟 kubelet
- 換 CNI 的話建議 kubeadm reset 整個 cluster 重來比較保險，手動清很容易漏

**你漏的部分：**
- **觀察性**：裝完能不能看到網路流量（Hubble、Kiali 那些）
- **效能基準**：用 iperf3 或類似工具測一下 Pod 之間的吞吐量，不同 CNI 可以比較
- **故障模擬**：砍掉一台 node 的 CNI pod，看 cluster 怎麼反應、會不會自動恢復
- **升級路徑**：CNI 怎麼升版、升版會不會斷網