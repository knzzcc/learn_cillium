**K8s Cluster 建置（約 30-40 分鐘）**
1. master 跑 kubeadm init，指定 pod network CIDR 跟 API server IP — 5 分鐘
2. 設定 kubectl config 讓一般使用者可以用 — 2 分鐘
3. worker1、worker2 跑 kubeadm join 加入 cluster — 5 分鐘
4. 確認三台 node 都 Ready — 等 CNI 裝完才會 Ready

[Demystifying Cilium: Learn How to Build an eBPF CNI Plugin from Scratch](https://www.solo.io/blog/cilium-build-ebpf-cni-plugin)
**Cilium 安裝（約 20-30 分鐘）**
1. 安裝 Cilium CLI — 2 分鐘
2. 用 cilium install 部署到 cluster — 5 分鐘
3. 等所有 Cilium pod 跑起來 — 5-10 分鐘
4. 跑 cilium status 跟 cilium connectivity test 驗證 — 10 分鐘
5. 裝 Hubble UI 觀察網路流量（可選）— 5 分鐘

[使用 K3s、GitHub 和 ArgoCD 實現 GitOps 自動化部署 - HackMD](https://hackmd.io/@jakiro-hans/k3s_argocd_github)
**ArgoCD 安裝（約 30-40 分鐘）**
1. 建 namespace、用 kubectl apply 裝 ArgoCD — 5 分鐘
2. 等 pod 全部跑起來 — 5 分鐘
3. 裝 argocd CLI — 2 分鐘
4. 用 NodePort 或 port-forward 把 UI 開出來 — 5 分鐘
5. 拿初始密碼登入 — 2 分鐘
6. 接一個 Git repo 部署一個測試 app 驗證 GitOps 流程 — 15-20 分鐘

PAT產生流程不熟，還有github action第一次用
Step 5 停在這

```yml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  # 1. 應用程式的名稱
  name: python-app
  # 2. ArgoCD Application CRD 必須建立在 ArgoCD 所在的命名空間
  namespace: argocd
spec:
  # 3. ArgoCD 專案，使用預設的 'default' 即可
  project: default

  # 4. 來源 (Source) - 您的 Git 儲存庫
  source:
    # 5. 替換成您的 "公開" manifests-repo URL
    repoURL: https://github.com/**`knzzcc`**/manifests-repo.git

    # 6. 要監控的 Git 分支
    targetRevision: main

    # 7. YAML 檔案所在的儲存庫路徑 ( '.' 代表根目錄)
    path: .

  # 8. 目的地 (Destination) - 您的 k3s 叢集
  destination:
    # 9. 'https://kubernetes.default.svc' 是 ArgoCD 內部的 API 伺服器地址
    server: https://kubernetes.default.svc

    # 10. 您希望將應用程式部署到 k3s 上的哪個命名空間
    #     這必須與 manifests-repo/deployment.yaml 中的 namespace 一致
    namespace: python-app-ns

  # 11. 同步策略 (Sync Policy)
  syncPolicy:
    # 12. 啟用自動同步
    automated:
      # (推薦) 當 Git 中的資源被刪除時，也自動從 k3s 中刪除
      prune: true
      # (推薦) 自動修復叢集狀態與 Git 狀態的偏差
      selfHeal: true

    # (可選) 確保 ArgoCD 在部署前建立 'python-app-ns' 命名空間
    syncOptions:
    - CreateNamespace=true

```


GitHub Actions / GitLab Runner 不是必須的。
這篇文章做的是完整的 CI/CD 流程：push code → 自動 build image → 更新 manifest → ArgoCD 自動部署。GitHub Actions 負責的是 **CI（build image + 更新 YAML）** 那段。

但 ArgoCD 本身只管 **CD（持續部署）**，它做的事就是監控 Git repo，發現 YAML 有變就自動同步到 cluster。所以你可以拆開用：

**只玩 ArgoCD 的話：**
1. 建一個 Git repo 放 K8s manifest（deployment.yaml、service.yaml）
2. ArgoCD 接上那個 repo
3. 你手動改 YAML、push 上去
4. ArgoCD 偵測到變更，自動部署

這樣完全不需要 GitHub Actions，你手動 push 就等於觸發部署。ArgoCD 的核心價值是「Git 上的 YAML = cluster 的實際狀態」，跟 CI 工具無關。

GitHub Actions 只是讓「build image → 更新 YAML」這段也自動化而已，學習階段先跳過沒差。

[吳東軒 - HackMD](https://hackmd.io/@wu-andy)  大量操作筆記
**Istio 安裝（約 1-1.5 小時）**
1. 裝 istioctl CLI — 2 分鐘
2. 用 istioctl install 選 demo profile — 5 分鐘
3. 等 istiod 跟 ingress gateway 跑起來 — 5-10 分鐘
4. 部署 BookInfo 範例應用 — 10 分鐘
5. 開啟 sidecar injection、設定 Gateway 跟 VirtualService — 10 分鐘
6. 裝 Kiali、Jaeger、Prometheus、Grafana 觀察性工具 — 10 分鐘
7. 測試 traffic management：routing、canary、fault injection — 20-30 分鐘
8. 測試 mTLS — 10 分鐘

Flannel也可以裝Istio
但有幾點你可能需要考慮：
- **性能損耗 (Double Encapsulation)**： 如果你在 Istio 中開啟了 **mTLS (雙向 TLS 加密)**，且 Flannel 也在底層進行了封裝（如 VXLAN），數據包會經過多次處理。這通常不會造成大問題，但在極高性能需求的環境下需要測試延遲。
    
- **CNI 的限制**： Flannel 本身功能較為單一，它**不支援 Network Policy**（即 Kubernetes 原生的防火牆規則）。如果你需要「Pod A 不能存取 Pod B」這種硬性隔離，通常會改用 **Calico** 或是在 Flannel 之上加掛 **Canal**。Istio 雖然也能做授權控制（AuthorizationPolicy），但它是基於應用層（L7）的。
    
- **MTU (最大傳輸單元)**： 因為 Flannel 的封裝（VXLAN）會佔用一些位元組，如果 MTU 設置不當，可能會導致 Istio 的大流量請求出現掉包或效能下降。

istioctl和cilliumctl只要在master裝就好了?
對的

worker1上面推薦裝甚麼?

 Claude responded: worker 上面不用額外裝什麼，它就是純粹的工作節點。
worker 上面不用額外裝什麼，它就是純粹的工作節點。

給我istio的建置操作指南

Bilibili
Linux网络命名空间核心 | 虚拟网线、虚拟交换机、虚拟路由器


Flux不太需要另外玩，ArgoCD有UI，Flux純CLI

ArgoCD 玩透，目標包含 App of Apps pattern、ApplicationSet、多環境部署、Sync Policy、Health Check、Rollback 這些
