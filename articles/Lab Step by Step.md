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


Bilibili
Linux网络命名空间核心 | 虚拟网线、虚拟交换机、虚拟路由器