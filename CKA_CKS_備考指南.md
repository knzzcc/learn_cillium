# CKA / CKAD / CKS 備考指南


[Kubernetes CKA 與 CKS 認證考試心得分享](https://blueswen.com/blog/2026/02/20/kubernetes-cka-%E8%88%87-cks-%E8%AA%8D%E8%AD%89%E8%80%83%E8%A9%A6%E5%BF%83%E5%BE%97%E5%88%86%E4%BA%AB/)

[AWS SAA 與 AWS SAP 認證考試心得分享](https://blueswen.com/blog/2025/12/14/aws-saa-%E8%88%87-aws-sap-%E8%AA%8D%E8%AD%89%E8%80%83%E8%A9%A6%E5%BF%83%E5%BE%97%E5%88%86%E4%BA%AB/)

[準備 CKS 的實戰心得分享](https://ithelp.ithome.com.tw/articles/10398957)

[CKA 2025 實戰經驗分享：DevOps日常的系統化學習之路](https://ithelp.ithome.com.tw/articles/10389777)

## 含金量排序

| 證照 | 含金量 | 目標族群 |
|------|--------|----------|
| **CKS** | 最高 | DevOps / SRE / Platform / Security Engineer |
| **CKA** | 高 | DevOps / SRE / Platform Engineer |
| **CKAD** | 中 | 偏開發者，證明會在 K8s 上部署應用 |

- CKAD 主要考 YAML 部署與應用操作，對有運維/架構背景的人 CP 值較低
- CKS 需要先持有 CKA 才能報考
- **建議路線：CKA → CKS**

---

## 定價與購買時機

### 目前定價（2026年）

| 項目 | 價格 |
|------|------|
| CKA 單買 | $445 USD |
| CKS 單買 | $445 USD |
| 兩張合計 | $890 USD |
| CKA + CKAD + CKS 三合一 bundle | $1,245 USD（等於 CKAD 免費送） |

> 目前沒有「CKA + CKS 兩合一」的專屬 bundle。

### 購買時機建議

| 時機 | 折扣幅度 |
|------|---------|
| **Black Friday / Cyber Monday**（每年 11 月）| 通常 40~50% off，最值得等 |
| Mega Sale（不定期） | 最高 50% off |
| 不定期 promo code | 約 30~35% off |

**Black Friday 預估價格：**
- 單張約 $220~$267 USD
- 兩張合計約 $445~$534 USD（可省 $350~$450）

**建議：** 等 11 月 Black Friday 再買。若也想考 CKAD，在折扣期間直接買三合一 bundle CP 值更高。

---

## 準備時間（有 K8s 工作經驗）

| 證照 | 預估時間 | 備註 |
|------|---------|------|
| CKA | 2~3 週 | 主要練速度和熟悉 kubectl 快捷操作 |
| CKAD | 1~2 週 | |
| CKS | 4~6 週 | 安全知識範圍較廣 |

> 考試重點不是知識量，而是**考場操作速度**。要練到能快速找官方文件並執行。

---

## CKS 需要額外教材

CKS 涵蓋的安全工具在日常工作中不一定常碰，需要額外準備。

### 必學工具 / 概念

- `Falco` — Runtime security（執行期威脅偵測）
- `Trivy` — Image vulnerability scan
- `OPA/Gatekeeper` 或 `Kyverno` — Admission control
- `AppArmor` / `seccomp` — 系統呼叫限制
- Network Policy 進階用法
- Supply chain security（image signing 等）
- RBAC 精細設定

### 推薦資源

| 資源 | 說明 |
|------|------|
| **Udemy - Kim Wüstkamp CKS 課程** | 最多人推薦，常特價 $15~20 |
| **KillerCoda** | 免費互動練習環境 |
| **Killer.sh** | 報名費附贈 2 次，最接近真實考試環境 |
| K8s 官方文件 | 考試可開啟，要能快速定位 |

---

## 其他值得考慮的證照

### AWS 系列

| 證照 | 含金量 | 說明 |
|------|--------|------|
| **AWS SAA** (Solutions Architect Associate) | 高 | 業界最認可的雲端入門證照，台灣幾乎是必備 |
| **AWS SAP** (Solutions Architect Professional) | 很高 | 進階架構設計，含金量明顯高於 SAA |
| **AWS ANS** (Advanced Networking - Specialty) | 高但偏利基 | 見下方說明 |
| AWS DevOps Professional | 高 | CI/CD + 自動化 + 監控，DevOps 職位吃香 |
| AWS SysOps Administrator | 中高 | 偏運維操作，與 K8s 背景互補 |

### AWS Advanced Networking - Specialty（ANS）

公認 AWS 最難的幾張之一，**偏利基**（niche）— 市場受眾窄，只有專門負責雲端網路架構的職位（Network Architect、Cloud Network Engineer）才特別在乎，但這類職缺薪資通常較高。

考試涵蓋：
- VPC 進階設計（Transit Gateway、VPC Peering、PrivateLink）
- Hybrid networking（Direct Connect、VPN、BGP 路由）
- DNS 進階（Route 53 Resolver、私有 hosted zone）
- Network security（Security Group、NACL、WAF、Shield、Network Firewall）
- Load Balancer 底層行為（ALB/NLB 差異、cross-zone）
- CloudFront / Global Accelerator 架構

> 學 Cilium / eBPF 的背景與 ANS 高度相關（BGP、路由、網路策略概念互通）。CKS + ANS 組合在金融、電信等安全性強的公司很少見、很吃香。

### 其他

| 證照 | 含金量 | 說明 |
|------|--------|------|
| **Terraform Associate** (HashiCorp) | 中高 | IaC 必備，準備簡單（1~2 週），幾乎每個 infra 職缺都會問 |
| **CiliumCert** (Isovalent) | 中，偏利基 | 學 Cilium 順手考，免費 |
| GCP / Azure 對應架構師證照 | 中高 | 公司有用對應雲才值得考 |

---

## 建議優先順序（台灣 / 亞洲 DevOps / SRE 市場）

```
CKA → AWS SAA → CKS → AWS SAP 或 AWS DevOps Pro
                            ↗
          Terraform Associate（中途可插入，準備簡單）
```

若職涯目標偏**網路架構**方向：
```
CKA → AWS SAA → CKS → AWS ANS
```
