態 Pod 形式運行 甚麼意思

晚上7:39





## Claude responded: 很好的問題！這是理解 Kubernetes 控制平面很重要的基礎概念。

很好的問題！這是理解 Kubernetes 控制平面很重要的基礎概念。

------

### 靜態 Pod（Static Pod）是什麼？

#### 先回顧普通 Pod 的建立方式

平常你建立一個 Pod，流程是這樣的：

```
你執行 kubectl apply
      ↓
API Server 收到請求
      ↓
寫入 etcd
      ↓
Scheduler 決定跑在哪個節點
      ↓
該節點的 kubelet 去拉映像、啟動容器
```

這個流程依賴 API Server、etcd、Scheduler 都正常運作。

------

#### 但有一個問題

**API Server 本身也是一個 Pod，那誰來啟動它？**

這就是雞生蛋、蛋生雞的問題——你沒辦法用 Kubernetes 來管理 Kubernetes 自己的核心元件。

------

#### 靜態 Pod 的解法

靜態 Pod 完全**繞過 API Server**，直接由 **kubelet 自己管理**。

kubelet 會持續監看節點上某個特定目錄（預設是 `/etc/kubernetes/manifests/`），只要這個目錄裡有 YAML 檔，kubelet 就直接啟動對應的容器，不需要經過任何其他元件：

```
/etc/kubernetes/manifests/
    ├── kube-apiserver.yaml        ← kubelet 看到這個
    ├── kube-controller-manager.yaml
    ├── kube-scheduler.yaml
    └── etcd.yaml
          ↓
      kubelet 直接啟動這些容器
      （不需要 API Server、不需要 etcd）
```

------

#### 靜態 Pod 的特性

| 特性                       | 說明                              |
| -------------------------- | --------------------------------- |
| 由誰管理                   | kubelet，不是 Kubernetes 控制平面 |
| 設定方式                   | 修改節點上的 YAML 檔案            |
| 重啟機制                   | 容器掛掉了，kubelet 自動重啟      |
| 能用 kubectl delete 刪除嗎 | 不行，刪了 kubelet 又會重新建立   |
| 出現在 kubectl get pods 嗎 | 會，但只是「鏡像」，無法真正控制  |

#### 這就是為什麼修改 kube-apiserver 的方式是「改 YAML 檔」

bash

```bash
# 不是用 kubectl edit，而是直接編輯節點上的檔案
vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

檔案一存檔，kubelet 偵測到變更，自動把舊的 API Server 容器關掉、用新設定重新啟動。這就是為什麼改完要等 1-2 分鐘——kubelet 需要時間偵測變更並重啟容器。