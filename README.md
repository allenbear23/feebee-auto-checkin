# Feebee 飛比價格 每日自動簽到與集點腳本

自動化執行「Feebee 飛比價格」每日簽到任務、錢包狀態查詢與點數變動監控，支援多帳號、GitHub Actions 雲端排程與 Bark iOS 推播通知。

---

## 核心原理

### 1. 舊版機制與失效原因
早期飛比簽到僅透過 Webview 請求 `https://feebee.com.tw/flybee/checkin` 即可入帳。自 2026 年底起，官方已將簽到邏輯完全收緊至 **App 原生層**，網頁端點雖仍回傳 HTTP 200，但伺服器端已不再歸戶點數。

### 2. 新版簽到逆向與驗證機制
新版簽到透過原生 API 端點完成：
- **端點網址**：`https://api.feebee.com.tw/v2/checkin`
- **關鍵請求標頭 (Headers)**：
  - `User-Agent`: 最新版 Android App 客戶端標頭（例如 `android-prod-release-v4.9.3_g-2017071429`）
  - `Channel`: `google`
  - `Package`: **App 原生簽到驗證 Token（即 `appu`）**
- **驗證機制**：
  若未帶 `Package` 或內容為空，伺服器回傳 `{"status":-1,"errno":205,"errmsg":"Package empty"}`。
  只有帶入有效的 `appu` Token，伺服器才會執行簽到並發放飛比點數（+2 點）。

### 3. 雙重流程保險
本專案整合兩階段執行鏈路：
1. **原生層簽到**：帶入 `Package: <appu>` 請求 `/v2/checkin`，直接觸發每日簽到入帳。
2. **任務中心與廣告回報**：發送 `reward_ad` (type=daily_checkin) 與 `backfill_ad` POST 請求，模擬完整 App 任務上報與抽獎流程。
3. **錢包狀態比對**：簽到前後查詢 `https://feebee.com.tw/member/point/wallet/`，確認點數增減（+2點）與入帳時間戳記。

---

## 憑證獲取方式 (Cookie 與 appu)

本腳本需要一組完整的 Cookie 字串，包含**網頁登入憑證**與 **App 原生 Token (`appu`)**。

### 步驟 1：取得網頁端 Cookie
1. 使用電腦或手機瀏覽器開啟 [飛比價格官網](https://feebee.com.tw) 並登入帳號。
2. 按 `F12` 打開開發者工具（DevTools），切換至 **Network (網路)** 標籤頁。
3. 重新整理頁面，點選任一前往 `feebee.com.tw` 的請求，複製其 **Request Headers** 內的 **Cookie**（包含 `u=...; h=...;` 等欄位）。

### 步驟 2：取得 App 原生 Token (`appu`)
1. 使用抓包工具（如 Charles、Fiddler、mitmproxy 或 HttpCanary）監聽手機「飛比價格 App」的流量。
2. 在 App 內進入「逛逛」頁面並點擊「簽到」。
3. 尋找請求 `https://api.feebee.com.tw/v2/checkin` 的封包。
4. 複製其 Request Header 中的 **`Package`** 值（格式為 `v2S-...` 的長字串）。

### 步驟 3：組合憑證
將上述兩者以分號加空格（`; `）合併為同一行字串：
```text
u=xxxxxxxx; h=xxxxxxxx; _ga=...; appu=v2S-aTZy...（長字串Token）
```

> **注意**：`appu` 具有時效性，若手機端重新登入、換機或伺服器憑證失效時，重新抓包取得更新即可。

---

## 本地執行說明

### 1. 環境需求
- Python 3.8 或以上版本（無需額外安裝第三方依賴庫，使用內建標準庫）。

### 2. 設定憑證
在專案根目錄下建立 `feebee_cookie.txt`，並填入剛才組合好的憑證：
```bash
# feebee_cookie.txt 內容
u=...; h=...; appu=v2S-...
```
> 如需多帳號，每行填入一個帳號的 Cookie，或以 `---` 隔開。

### 3. (可選) 設定 Bark 推播
若需推播通知，可建立 `bark_key.txt`，內容填入您的 Bark Device Key 或完整 URL：
```text
https://api.day.app/YOUR_KEY/
```

### 4. 執行腳本
```bash
python3 feebee_checkin.py
```

---

## GitHub Actions 雲端自動排程

本專案已內建 GitHub Actions 工作流程（`.github/workflows/feebee_checkin.yml`），每日台灣時間凌晨 `00:08` 自動執行。

### 設定步驟：
1. 將本專案 Fork 或 Push 至您的私有 GitHub Repository。
2. 進入 Repository 的 **Settings** -> **Secrets and variables** -> **Actions**。
3. 點擊 **New repository secret**，新增以下 Secret：
   - **`FEEBEE_COOKIE`**（必填）：填入組合好的完整 Cookie 字串（含 `appu=...`）。多帳號可用換行或 `---` 分隔。
   - **`BARK_KEY`**（選填）：填入您的 Bark Key 或 Webhook URL。
4. 設定完成後，可至 **Actions** 標籤頁點擊 **Feebee Daily Check-in** -> **Run workflow** 進行手動觸發測試。

---

## 授權與免責聲明
本專案僅供程式開發、個人研究與 API 協定分析測試使用。請勿用於批量註冊或大規模濫用行為。使用者需自行承擔帳號安全與平台服務條款之風險。
