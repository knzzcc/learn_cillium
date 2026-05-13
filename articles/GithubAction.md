GitHub Actions 其實就一個核心概念：**事件觸發工作流程**。

**思維模型**

你的 repo 發生了某件事（push、PR、排程、手動觸發）→ GitHub 開一台臨時的 VM → 按照你寫的步驟跑指令 → 跑完 VM 銷毀。就這樣，本質上就是一台免費的、用完即丟的 Linux/Windows 機器幫你做事。

**核心結構**

一個 `.github/workflows/xxx.yml` 檔案就是一個 workflow，裡面三層：

- **Trigger（on）** — 什麼時候跑：push、pull_request、schedule、workflow_dispatch（手動）
- **Job** — 跑在哪種機器上，多個 job 預設平行跑
- **Step** — 每一步做什麼，可以是一行 shell 指令，也可以是別人寫好的 action（`uses: actions/checkout@v4`）

**常見用途**

- CI：push 之後自動跑測試、lint、build
- CD：merge 到 main 之後自動部署到 staging/production
- Docker：build image 然後 push 到 registry
- 排程任務：每天跑一次爬蟲、清理、通知
- PR 自動化：自動加 label、跑 code review 檢查、跑安全掃描
- Release：打 tag 自動 build 然後發 release

**幾個重要觀念**

每個 job 是獨立的乾淨環境，不共享檔案。Job 之間要傳東西用 artifact 或 cache。Secrets 放在 repo settings 裡，workflow 裡用 `${{ secrets.XXX }}` 引用，不要寫死在 yml 裡。`actions/checkout` 幾乎每個 workflow 第一步都要加，不然 runner 上沒有你的程式碼。

**除錯方式**

最直接的就是看 Actions tab 裡每個 step 的 log，點開就能看 stdout/stderr。如果 log 不夠詳細，在 step 裡面加 `run: env` 或 `run: ls -la` 印出環境狀態。想要更多 debug 資訊，在 repo secrets 加 `ACTIONS_RUNNER_DEBUG = true`，會印出 runner 層級的 debug log。本地測試可以用 `act` 這個工具，它會在 Docker 裡模擬 GitHub runner 跑你的 workflow，不用每次 push 上去等。

**踩坑提醒**

yml 縮排錯就整個壞掉，跟你之前 netplan 一樣。分支名稱要對，`on: push: branches: [main]` 跟你實際的分支名要一致。Free tier 每月有 2000 分鐘額度，跑太多會超過。Self-hosted runner 可以跑在你自己的機器上，不受額度限制，這個之後可以跑在你的 K8s cluster 上。