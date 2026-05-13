# Istio 建置操作指南

> 環境前提：K8s cluster 已建好，三台 node 都 Ready，CNI（Cilium 或 Calico）已安裝

---

## 一、安裝 Istio

### 1.1 下載 istioctl

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH="$PWD/bin:$PATH"

# 加到 bashrc 避免每次都要 export
echo 'export PATH="$HOME/istio-*/bin:$PATH"' >> ~/.bashrc

# 確認
istioctl version
```

### 1.2 安裝 Istio 到 cluster

```bash
# 用 demo profile，包含所有功能，適合學習
istioctl install --set profile=demo -y
```

### 1.3 驗證安裝

```bash
kubectl get pods -n istio-system
kubectl get svc -n istio-system

# 應該看到 istiod、istio-ingressgateway、istio-egressgateway 都 Running
```

---

## 二、啟用 Sidecar 自動注入

```bash
# 對 default namespace 啟用自動注入
kubectl label namespace default istio-injection=enabled

# 確認 label
kubectl get namespace default --show-labels
```

之後在 default namespace 建立的 Pod 都會自動注入 Envoy sidecar proxy。

---

## 三、部署 BookInfo 範例應用

### 3.1 部署應用

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

# 等 pod 跑起來
kubectl get pods

# 每個 pod 應該顯示 2/2（app + sidecar）
```

### 3.2 驗證應用

```bash
# 確認服務都起來了
kubectl get svc

# 測試應用
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" \
  -c ratings -- curl -s productpage:9080/productpage | head -20
```

---

## 四、設定 Ingress Gateway

### 4.1 建立 Gateway 和 VirtualService

```bash
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

# 確認
kubectl get gateway
kubectl get virtualservice
```

### 4.2 取得存取地址

```bash
# 取得 ingress gateway 的 NodePort
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway \
  -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')

export INGRESS_HOST=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')

export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

echo "http://$GATEWAY_URL/productpage"
```

從 Windows 瀏覽器打開上面的地址，應該看到 BookInfo 頁面。

---

## 五、安裝觀察性工具

### 5.1 Kiali、Jaeger、Prometheus、Grafana

```bash
kubectl apply -f samples/addons/
kubectl rollout status deployment/kiali -n istio-system
```

### 5.2 存取 Kiali Dashboard

```bash
# 方法一：port-forward
kubectl port-forward svc/kiali -n istio-system 20001:20001 --address 0.0.0.0

# 從 Windows 瀏覽器開 http://192.168.137.101:20001
```

```bash
# 方法二：改成 NodePort（比較方便）
kubectl patch svc kiali -n istio-system -p '{"spec": {"type": "NodePort"}}'
kubectl get svc kiali -n istio-system
# 看 NodePort 的 port，從 Windows 用 http://master的IP:NodePort 存取
```

### 5.3 產生流量讓 Dashboard 有資料

```bash
for i in $(seq 1 100); do
  curl -s -o /dev/null "http://$GATEWAY_URL/productpage"
done
```

回到 Kiali → Graph → 選 default namespace，就能看到服務之間的流量圖。

---

## 六、流量管理

### 6.1 建立 Destination Rules

```bash
kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml

kubectl get destinationrules
```

### 6.2 路由到指定版本（全部走 v1）

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml

# 現在刷新頁面，reviews 只會顯示沒有星星的版本（v1）
```

### 6.3 依使用者路由（Canary）

```bash
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml

# 用 jason 登入 → 看到黑色星星（v2）
# 其他人 → 看到沒有星星（v1）
# 帳號：jason，密碼：隨便填
```

### 6.4 流量權重分配

```bash
# 50% v1, 50% v3
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v3
      weight: 50
EOF
```

刷新頁面多次，會交替看到無星星（v1）和紅色星星（v3）。

---

## 七、故障注入

### 7.1 注入延遲

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 100
        fixedDelay: 7s
    route:
    - destination:
        host: ratings
        subset: v1
EOF

# 現在打開 productpage 會發現 reviews 區塊要等很久
```

### 7.2 注入錯誤

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      abort:
        percentage:
          value: 50
        httpStatus: 500
    route:
    - destination:
        host: ratings
        subset: v1
EOF

# 50% 的請求會回傳 500 錯誤
```

---

## 八、熔斷 Circuit Breaking

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
EOF
```

用 fortio 壓測觀察熔斷效果：

```bash
kubectl apply -f samples/httpbin/sample-client/fortio-deploy.yaml

FORTIO_POD=$(kubectl get pods -l app=fortio -o jsonpath='{.items[0].metadata.name}')

# 送 20 個併發請求
kubectl exec "$FORTIO_POD" -c fortio -- \
  fortio load -c 3 -qps 0 -n 20 -loglevel Warning \
  http://productpage:9080/productpage
```

觀察結果中有多少 200 和多少 503，503 就是被熔斷擋掉的。

---

## 九、mTLS 雙向 TLS

### 9.1 確認 mTLS 狀態

```bash
# Istio demo profile 預設就是 PERMISSIVE 模式
# 表示同時接受明文和加密流量

# 檢查目前的 PeerAuthentication
kubectl get peerauthentication --all-namespaces
```

### 9.2 啟用 STRICT mTLS

```bash
cat <<EOF | kubectl apply -f -
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
EOF

# 現在所有 mesh 內的流量都強制加密
```

### 9.3 驗證

```bash
# 從有 sidecar 的 pod 連 → 成功
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" \
  -c ratings -- curl -s productpage:9080/productpage | head -5

# 從沒有 sidecar 的 namespace 連 → 應該失敗
kubectl create namespace no-mesh
kubectl run test --image=curlimages/curl -n no-mesh --rm -it --restart=Never \
  -- curl -s productpage.default:9080/productpage
```

---

## 十、清理

```bash
# 刪除 BookInfo
kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl delete -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl delete -f samples/bookinfo/networking/destination-rule-all.yaml
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml

# 刪除觀察性工具
kubectl delete -f samples/addons/

# 移除 Istio
istioctl uninstall --purge -y
kubectl delete namespace istio-system
kubectl label namespace default istio-injection-
```

---

## 重點筆記

- **profile 選擇**：demo 適合學習（全開），default 適合生產（精簡）
- **sidecar injection**：一定要先 label namespace，不然 pod 不會注入 proxy
- **流量管理順序**：先建 DestinationRule 定義 subset，再建 VirtualService 做路由
- **除錯指令**：
  - `istioctl analyze` — 檢查設定有沒有問題
  - `istioctl proxy-status` — 看所有 proxy 的同步狀態
  - `istioctl proxy-config routes <pod>` — 看某個 pod 的路由設定
  - `kubectl logs <pod> -c istio-proxy` — 看 sidecar 的 log
