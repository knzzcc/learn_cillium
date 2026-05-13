# KEDA

**Kubernetes Event-Driven Autoscaling**

讓 Pod 根據事件數量自動擴縮。

---

## 問題背景

原生 HPA（Horizontal Pod Autoscaler）只能根據 CPU / Memory 擴縮，無法感知業務層的事件。

## KEDA 擴展的觸發來源

- Kafka topic lag 量
- Redis queue 長度
- RabbitMQ 訊息數
- Azure Service Bus、AWS SQS
- Prometheus 的任意 metric
- Cron 時間表

## 例子

Consumer 在處理 Kafka 訊息，訊息堆積超過 1000 條就自動開更多 Pod，處理完縮回 0。原生 HPA 做不到。

## 定位

應用層的彈性伸縮，不是網路層的東西。跟 Cilium / Istio 不在同一個層面。
