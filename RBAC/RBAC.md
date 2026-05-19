RBAC 的核心其實很簡單，只有四個東西要記。先看這張圖：

V

visualize

V

visualize show_widget



<iframe title="visualize: K8s rbac core model" sandbox="allow-scripts allow-same-origin allow-forms" allow="fullscreen *; clipboard-write *" src="https://078622d42da62e6e61c7b0fd96e23c75.claudemcpcontent.com/mcp_apps?connect-src=https%3A%2F%2Fesm.sh+https%3A%2F%2Fcdnjs.cloudflare.com+https%3A%2F%2Fcdn.jsdelivr.net+https%3A%2F%2Funpkg.com&amp;resource-src=https%3A%2F%2Fesm.sh+https%3A%2F%2Fcdnjs.cloudflare.com+https%3A%2F%2Fcdn.jsdelivr.net+https%3A%2F%2Funpkg.com+https%3A%2F%2Fassets.claude.ai&amp;dev=true" style="width: 100%; border: none; background-color: transparent; height: 520px;"></iframe>

RBAC 整個機制就是在回答一句話：**「誰」能對「什麼資源」做「什麼操作」**。拆開來只有三個角色：

**Subject（誰）** 有三種身份。`User` 是真人（開發者、管理員），K8s 本身不管帳號，靠外部認證（公司的 SSO、憑證等）。`Group` 是一群 User 的集合。`ServiceAccount` 是給 Pod 裡跑的程式用的身份——你的應用要呼叫 K8s API 時就靠它。

**Role（能做什麼）** 是一組權限規則。每條規則寫明三件事：`apiGroups`（哪個 API 群組，例如核心的 `""` 或 `apps`）、`resources`（哪種資源，例如 `pods`、`deployments`）、`verbs`（允許的動作，例如 `get`、`list`、`create`、`delete`）。

**RoleBinding（黏在一起）** 把 Subject 和 Role 綁起來。沒有 RoleBinding，Role 只是一張寫好的規則紙，誰都用不了。

然後是作用範圍的問題。`Role` + `RoleBinding` 只在一個 namespace 裡生效——你在 `dev` namespace 給某人權限，他碰不到 `prod` 的東西。如果需要跨所有 namespace 或管理 namespace 本身、node 這類叢集層級的資源，就要用 `ClusterRole` + `ClusterRoleBinding`。

舉個具體例子幫你感受：

假設你想讓開發者小明在 `dev` namespace 裡能看 Pod 和 log 但不能刪東西，你會建一個 Role 寫 `verbs: ["get", "list", "watch"]`，再建一個 RoleBinding 把小明綁上去。如果小明調去當 SRE 需要看所有 namespace 的 Pod，你就改用 ClusterRole + ClusterRoleBinding。

點圖上的元件可以看更多細節，或者告訴我你想看具體的 YAML 範例。

## You said: dev和SRE能看到的權限不一樣 還有嗎?

dev和SRE能看到的權限不一樣 還有嗎?

下午6:29

## Claude responded: 好問題，實務上常見的角色不只這兩個。讓我把典型的角色權限分級整理出來： 從上到下看，權限是逐層縮小的：

好問題，實務上常見的角色不只這兩個。讓我把典型的角色權限分級整理出來：

V

visualize

V

visualize show_widget



<iframe title="visualize: K8s rbac role tiers" sandbox="allow-scripts allow-same-origin allow-forms" allow="fullscreen *; clipboard-write *" src="https://078622d42da62e6e61c7b0fd96e23c75.claudemcpcontent.com/mcp_apps?connect-src=https%3A%2F%2Fesm.sh+https%3A%2F%2Fcdnjs.cloudflare.com+https%3A%2F%2Fcdn.jsdelivr.net+https%3A%2F%2Funpkg.com&amp;resource-src=https%3A%2F%2Fesm.sh+https%3A%2F%2Fcdnjs.cloudflare.com+https%3A%2F%2Fcdn.jsdelivr.net+https%3A%2F%2Funpkg.com+https%3A%2F%2Fassets.claude.ai&amp;dev=true" style="width: 100%; border: none; background-color: transparent; height: 989px;"></iframe>

從上到下看，權限是逐層縮小的：

**Cluster admin** 是上帝視角，什麼都能做，包括管理 RBAC 本身。實務上只有平台負責人才會拿到，而且最好搭配 audit log 監控誰用了這個權限。

**SRE / DevOps** 需要跨所有 namespace 看全局來排查問題，所以用 ClusterRole。但關鍵是不給他們改 RBAC 的權限——不然等於是變相的 admin。

**Developer** 只需要管自己團隊的 namespace，用 Role 綁在特定 namespace 就好。`dev` 和 `staging` 可以給寬一點，`prod` 則收緊甚至只給唯讀。

**Viewer / QA** 只需要看狀態、看 log，verbs 只開 `get`、`list`、`watch`，連 `create` 都不給。

**ServiceAccount** 是最容易被忽略但最重要的。很多人部署應用時沒設定，Pod 就用了 default ServiceAccount，可能拿到不該有的權限。正確做法是為每個應用建立專屬的 ServiceAccount，只給它真正需要的最小權限。

你可以點每個角色下方的連結看具體的 YAML 長什麼樣子。有想看哪個角色的實際設定嗎？