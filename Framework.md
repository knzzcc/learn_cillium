# 學習、寫作、找資料框架

## 學習框架

1. 掌握核心觀念
2. 查核驗證
3. 動手實驗
4. 學 debug
5. 搞破壞 / 極端狀況

---

## 入場框架（進入陌生領域時）

不熟悉一個領域時，問題是**不知道地圖長什麼樣**，不是知識不夠。

解法：先找全景圖，再找細節。

1. **找 threat model 或 attack taxonomy** — 例如 MITRE ATT&CK for Containers，看完就知道這個領域有哪些維度
2. **找公認的入門資源** — 某個 repo、conference talk、書
3. **找 3-5 個這個領域的人在寫什麼** — 看他們最近在討論什麼問題

> 找到 re0-kubernetes-sec-archive 就是做對了這一步。

---

## 寫作框架

核心問題：解決了什麼問題、怎麼做到的、可能延伸的問題

具體結構：

1. **為什麼這件事重要** — 讀者為什麼要在意
2. **大多數人的誤解是什麼** — 製造張力
3. **實際怎麼運作** — 核心內容
4. **lab 驗證** — 截圖、指令、結果
5. **防禦或延伸** — 讓讀者帶走一個行動

---

## 找資料框架

| 目的 | 去哪找 |
|---|---|
| 搞清楚地圖 | awesome-xxx repo、CNCF landscape、MITRE ATT&CK |
| 找權威說法 | 官方文件、KubeCon talk |
| 找踩坑經驗 | GitHub issue、HackMD、個人部落格 |
| 找最新動態 | Twitter/X 追關鍵人物、CNCF blog |
| 找攻擊手法 | freebuf、seebug、conference paper |
