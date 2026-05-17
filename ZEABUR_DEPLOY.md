# Zeabur 部署步驟

針對 hermes-agent 跑 **Telegram messaging gateway**（24/7 long polling）的部署指引。
不需要 domain、不需要 expose public port — Telegram bot 走 outbound polling。

---

## 前置

- 在 Zeabur 已建立 project（從這份文件出發是: https://zeabur.com/projects/6a09d2d0a9aa641e8597fbf1）
- 手上已有**新的**（未洩漏的）：
  - OpenRouter API key
  - Telegram bot token（@BotFather）
  - 自己的 Telegram user ID（@userinfobot）

---

## Dashboard 操作（依序點）

### 1. 連 GitHub repo

1. 進 Zeabur project → `Add Service` → `Deploy from GitHub`
2. 第一次會跳轉 GitHub 授權 Zeabur App
   - 授權範圍：選 `Only select repositories` → 勾 `tzangms/hermes-agent`（不要 All repositories）
3. Repo 列表選 `tzangms/hermes-agent`
4. Branch 選 `zeabur-deploy`（不是 `main`，main 沒有 Zeabur 設定檔）

Zeabur 偵測到 root 的 `Dockerfile` + `zbpack.json` 後會用 Dockerfile builder。

### 2. 設環境變數

進 Service → `Variables` tab，加入：

| Key | Value | 必填 |
|---|---|---|
| `OPENROUTER_API_KEY` | （新發的 OpenRouter key） | ✅ |
| `TELEGRAM_BOT_TOKEN` | （新發的 bot token） | ✅ |
| `TELEGRAM_ALLOWED_USERS` | 你的 Telegram user ID | ✅ **不填 = 對全世界開放** |

可選增強：

| Key | Value | 用途 |
|---|---|---|
| `TELEGRAM_HOME_CHANNEL` | 你跟 bot 的 chat_id | cron 排程預設送哪個對話 |
| `HERMES_MAX_ITERATIONS` | `60` | 單次回應最多 LLM 呼叫次數，防失控燒 quota |

### 3. 加 Persistent Volume

Service → `Volumes` tab → `Add Volume`：

- Mount path: `/opt/data`
- Size: `5 GB`（起跳；若會累積很多對話歷史可以更大）

**為什麼必要**：對話歷史、user model、skills、cron 排程、Telegram session 全部存這裡。沒有 volume → 每次 restart 全部丟失，每次冷啟動都要重新 lazy install Python deps。

### 4. 不要設 Domain / Public Port

- Telegram gateway 用 **long polling**（純 outbound 到 `api.telegram.org`），不需 inbound port
- Zeabur 預設給 service 開 domain，這個服務**不要**啟用，避免不必要的 endpoint 暴露

### 5. Deploy

按 `Deploy` 觸發 build。

**預期時間**：
- Build: 15–25 分鐘（首次；裝 Playwright Chromium + Node + Python deps）
- Image size: 約 4.2 GB
- Cold start: image pull + container boot 約 1–3 分鐘
- 之後 redeploy 有 layer cache，會快很多

---

## 驗證 Boot 成功

Service → `Logs` → `Runtime`，找這幾行（從本機測試出來的順序）：

```
gateway.run: Starting Hermes Gateway...
tools.lazy_deps: Lazy-installing python-telegram-bot[webhooks]==22.6 for feature 'platform.telegram'
gateway.run: Connecting to telegram...
gateway.platforms.telegram: [Telegram] Connected to Telegram (polling mode)
gateway.run: ✓ telegram connected
gateway.run: Gateway running with 1 platform(s)
gateway.run: Cron ticker started (interval=60s)
```

看到 `✓ telegram connected` 就代表 platform 起來了。

如果看到的不是這個 — 把 log 貼回對話，我們一起 debug。

---

## End-to-End 測試

1. 在 Telegram 找你的 bot（BotFather 給你的 username，例如 `@my_hermes_bot`）
2. 傳「hello」
3. 等 10–30 秒（首次回應慢，要 cold-load OpenRouter）
4. Bot 應該回應一段話

如果沒回應：
- 是否漏設 `TELEGRAM_ALLOWED_USERS` 或填錯 user ID？bot 只回應 allowed user
- OpenRouter key 是否有額度？
- 看 dashboard log 是否有 error

---

## 日常維護

| 動作 | 怎麼做 |
|---|---|
| 看即時 log | Dashboard → Service → Logs |
| 重啟 | Dashboard → Service → Restart |
| 換 env | Dashboard → Variables → 改完按 Save，會自動 redeploy |
| 升級 hermes | 在本機 `git fetch upstream && git rebase upstream/main && git push origin zeabur-deploy --force-with-lease`，Zeabur 偵測新 commit 自動 build |
| 停掉 | Dashboard → Service → Suspend（保留設定）或 Delete |

---

## 預期成本

按 Zeabur 撰寫時的計費：
- 4.2 GB image + persistent volume + always-on container ≈ **$10–30/月**（看你選的 plan、CPU/memory 配額）
- 加上 OpenRouter 用量（依互動頻率）

如果使用率很低（一天傳幾句話），會覺得 always-on container 是浪費。屆時值得評估改用 hermes 官方支援的 **Daytona / Modal** serverless backend（idle 時近乎不收費）。

---

## 故障排查

| 症狀 | 可能原因 |
|---|---|
| Build 卡在 `npm install` 或 `uv sync` | Zeabur build resource 不足，升 plan |
| Build 失敗 OOM | 同上 |
| Runtime log 沒看到 `Connecting to telegram` | `TELEGRAM_BOT_TOKEN` 沒設或設錯，Variables 確認後 redeploy |
| Bot 不回應你 | `TELEGRAM_ALLOWED_USERS` 沒含你的 user ID |
| 每次 restart 全部對話消失 | `/opt/data` 沒掛 persistent volume |
| GitHub repo 選不到 | https://github.com/settings/installations → Zeabur → Configure → 加上 `tzangms/hermes-agent` |
