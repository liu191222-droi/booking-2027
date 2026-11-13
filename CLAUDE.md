# 2027 預約系統（員工登入）— GitHub Pages 版

員林基督教醫院藥劑課的「生日體檢及其它假別預約系統」。
網址：https://liu191222-droi.github.io/booking-2027/
推送到 `main` 後，GitHub Pages 約 1～2 分鐘自動更新。

使用者以繁體中文溝通，回覆與介面文字一律用繁體中文。

## 檔案

| 檔案 | 說明 |
|---|---|
| `index.html` | 主程式：登入、預約、後台管理、大夜藥師排班。單一檔案，內含 CSS、JS 與 base64 圖片（有兩行超長的圖片資料，讀檔時用 offset/limit 或 grep，不要整份讀） |
| `overtime.html` | 加班費申請，以 iframe 嵌在主頁內，透過 `window.parent.OTHOST` 取得登入者、員工名單與資料庫，本身不連資料庫 |
| `parttime.html` | 兼職藥師排班系統，以 iframe 嵌在主頁「🕒 兼職藥師排班」（限人員類別「兼職藥師」與管理者）。透過 `window.parent.PTHOST` 取得登入者與資料庫，本身不連資料庫；寫入前要轉成主頁面的物件（跨 iframe 的物件 Firestore 不接受）。原始版本在 claude.ai Artifact（https://claude.ai/artifact/FgjtuKqhSkMgsseM4NrryZ），同步時要保留檔案中的 PTHOST 轉接段落 |
| `firestore.rules` | Firestore 存取規則的備份。實際規則在 Firebase 主控台設定，改了這個檔案要提醒使用者到主控台貼上並發布 |

## 架構重點（修改前必讀）

- **原始版本在 claude.ai Artifact**（https://claude.ai/artifact/16vhxeQpbpcYQBZXvKZhaR），使用 Claude 的 `claude.use("db")` 與 `claude.use("downloads")`。
- **GitHub 版在 `index.html` 的 xlsx `<script>` 之前多了一段 Firebase 轉接程式**：載入 firebase 10.12.2 compat SDK、`FIREBASE_CONFIG`，並定義 `window.claude.use()`：
  - `"db"` → 匿名登入後回傳 `firebase.firestore()`（API 與原本的 db 相容：`doc().set/onSnapshot/delete`、`collection().doc()`、`snap.docs`、`s.exists`）
  - `"downloads"` → 用 Blob + `<a download>` 下載
- **這段轉接程式與 `FIREBASE_CONFIG` 絕對不能被覆蓋或刪除**，否則網站會變成「單機模式」，沒有人能登入。
- 從 claude.ai Artifact 同步新版本時：用 Artifact 工具讀取最新版 → 去掉發佈時自動加上的外層 HTML 殼（從內層 `<!DOCTYPE html>` 開始）→ 重新插入 Firebase 轉接程式 → 把提到「登入 claude.ai」的提示文字換成「無法連線到資料庫」之類的說法 → 檢查錯誤代碼判斷同時接受 `permission_denied` 與 Firebase 的 `permission-denied`。`overtime.html` 可以直接同步。

## 資料（Firestore，專案 `booking-2027-9ee88`）

集合：`bookings`（預約）、`config`（`main` 系統設定，含 `holidays` 全年度行事曆／醫院公休日、`calpdf` 行事曆 PDF（base64，由後台上傳，不放進儲存庫）、`staff` 員工名單、`close_YYYY-MM` 月份關閉）、`pw`（密碼雜湊）、`avail`（大夜班填寫）、`sched`（大夜班表）、`ot`（加班資料）、`ptconfig`／`ptstaff`／`ptsched`／`ptmyshifts`（兼職藥師排班）。
規則只允許已匿名登入的使用者讀寫這些集合；新增集合時要同時更新 `firestore.rules` 並請使用者到主控台發布。

## 安全規則

- **員工名單只存在資料庫 `config/staff`**（格式 `員編|姓名|班別|類別`），不要寫回程式碼。這是公開儲存庫。
- 不要把任何密碼或預設密碼寫進程式碼。
- 系統預設管理者員編在 `index.html` 的 `ADMINS` 常數；其他管理者由後台「帳號管理」指定，存在 `config/main` 的 `admins`。
- `admin` 帳號（最高權限）的密碼只存雜湊在 `pw/admin`，由後台設定；管理者不能修改或停用 admin。
- 員工登入只在前端檢查，懂技術的人可以直接讀取 Firestore 資料。這是已知限制，使用者已知悉。

## 測試

這台雲端機器連不到 cdn.jsdelivr.net 與 github.io。測試方式：
1. `npm pack firebase@10.12.2` 解壓縮取得 compat SDK
2. 在本機用 `python3 -m http.server` 提供頁面
3. 用 Playwright 開啟頁面，把 jsdelivr 的請求導向本機 SDK 檔案，瀏覽器的 proxy 設定 `bypass: 'localhost,127.0.0.1'`
4. 登入測試時不要修改真實資料；需要寫入測試時先告知使用者
