這是 Kubernetes 儲存架構中的兩個重要概念。

**In-tree CSI** 指的是直接內建在 Kubernetes 核心程式碼（k8s repo）裡的儲存驅動，例如早期的 AWS EBS、GCE PD、NFS 等 volume plugin。它們跟 Kubernetes 本體一起編譯、一起發布，所以叫「in-tree」（在原始碼樹裡面）。

**Out-of-tree CSI** 則是透過 CSI（Container Storage Interface）標準介面，以獨立的外部元件（通常是一組 Pod/容器）部署的儲存驅動。它們有自己的 repo、自己的發布週期，不需要改動 Kubernetes 核心程式碼。

為什麼要從 in-tree 遷移到 out-of-tree？主要幾個原因：

- in-tree plugin 綁死在 K8s 發布週期，存儲廠商要修 bug 或加功能得等 K8s 新版本出來，非常慢。
- 所有 in-tree plugin 的程式碼都塞在 kubelet 和 controller-manager 裡，讓核心二進位越來越肥。
- 不同儲存廠商的 plugin 品質參差不齊，卻全部耦合在核心裡，增加維護負擔。

CSI 就是為了解決這些問題而設計的標準介面，讓儲存驅動可以獨立開發、獨立部署。目前社群的方向是逐步把所有 in-tree plugin 遷移（migration）到 out-of-tree CSI driver，in-tree 的最終會被移除