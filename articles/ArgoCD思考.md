
ArgoCD除了k8s cd之外 還有甚麼玩法


ArgoCD 核心就是 GitOps CD，但可以延伸玩很多東西：

**多環境管理** 一個 Git repo 裡用 Kustomize overlay 或 Helm values 區分 dev / staging / prod，ArgoCD 用不同 Application 指向不同資料夾或 branch，模擬真實的多環境部署流程。

**App of Apps 模式** 一個 Application 管理其他 Application，整個 cluster 的所有服務用一個 root app 就能一次部署起來。這是管理大量微服務的標準做法。

**ApplicationSet** 搭配 generator（Git、List、Cluster），自動根據規則批量產生 Application。比如 Git repo 裡每新增一個資料夾就自動建一個 app，不用手動建。

**搭配 Sealed Secrets 或 SOPS** 練習怎麼把 secret 安全地放進 Git，讓 ArgoCD 部署的時候自動解密。這在實際環境很重要。

**Image Updater** ArgoCD Image Updater 可以監控 container registry，有新 image 就自動更新 Git repo 裡的 tag，觸發部署。等於把 CI 的最後一哩路也自動化了。

**Rollback 跟 Sync Policy** 玩自動 sync、self-heal（有人手動改 cluster 裡的東西會自動修回去）、prune（Git 刪掉的資源 cluster 也自動刪）。故意搞壞東西看 ArgoCD 怎麼修復。

**搭配 Istio 玩漸進式部署** ArgoCD + Argo Rollouts + Istio，做 canary deployment 根據流量比例慢慢切換版本，這是面試很愛問的組合。

你的環境三台 node 夠玩這些了。

ArgoCD 跟ansible terraform???

## Claude responded: 三個定位完全不同，不衝突：

三個定位完全不同，不衝突：

**Terraform** → 建基礎設施 VM、VPC、Load Balancer、RDS 這些。回答「我要什麼環境」。跑完之後就不管了，除非你要改基礎設施。

**Ansible** → 設定機器 裝軟體、改設定檔、設 user、部署 agent。回答「機器裝好之後怎麼設定」。跑完也不管了，除非你要重新設定。

**ArgoCD** → 部署應用到 K8s 把 Git repo 裡的 YAML 同步到 cluster，持續監控。回答「K8s 上面跑什麼」。它是一直在跑的，持續確保 cluster 狀態跟 Git 一致。

**實際流程串起來：**

```
Terraform 開 VM / 雲端資源
    ↓
Ansible 裝 K8s、設定 OS
    ↓
ArgoCD 部署 app 到 K8s，持續同步
```

用你現在的環境對應：

- **Terraform 的位置** → 你用 vmrun script 建 VM（因為 Workstation 不好用 Terraform）
- **Ansible 的位置** → 三台裝 containerd、kubeadm 那些步驟，可以用 Ansible playbook 自動化
- **ArgoCD 的位置** → K8s 跑起來之後，app 的部署交給它

所以不是選一個用，是三個各管一層。面試的時候能講清楚這個分層就很加分。


所以通常terraform ansible是額外的repo在維護的???

對，通常就是分開的 repo：

```
infra-terraform/          ← 基礎設施
├── modules/
├── environments/
│   ├── dev/
│   ├── staging/
│   └── prod/
└── main.tf

infra-ansible/            ← 機器設定
├── inventory/
├── roles/
│   ├── common/
│   ├── containerd/
│   └── kubeadm/
└── playbooks/

k8s-manifests/            ← ArgoCD 監控這個
├── apps/
│   ├── frontend/
│   ├── backend/
│   └── monitoring/
├── base/
└── overlays/
    ├── dev/
    ├── staging/
    └── prod/
```

分開的原因：

**權限不同** — 動 Terraform 的人通常是 infra team，動 k8s-manifests 的是 dev team，不該混在一起。

**變更頻率不同** — Terraform 可能幾個月才改一次，Ansible 偶爾跑，但 app 部署可能一天好幾次。

**審核流程不同** — 改基礎設施的 PR 要嚴格審，改 app 的 image tag 可以快速過。

**觸發方式不同** — Terraform 跟 Ansible 是手動或 CI 觸發跑一次就結束，ArgoCD 是持續監控自動同步。

小公司或 side project 可能全塞一個 repo，但只要規模一大就一定會拆開。