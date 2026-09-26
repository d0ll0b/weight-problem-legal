# Google Play Data safety 表單填寫對照

> 依 2026-09-22 的程式碼實況整理(版本 1.1.0+2)。
> Play Console 的欄位文字偶有調整,以下以「問題語意」對照,填的時候照語意找。
> **每次改動同步範圍或新增 SDK,這份要一起更新** —— 申報不實是下架等級的問題。

## 判斷依據(程式碼事實)

| 事實 | 出處 |
|---|---|
| 同步的集合只有這 5 個 | `SyncService._collections`:`exercises`(僅自訂)、`routines`、`sessions`、`bodyWeights`、`dietEntries` |
| `AppSettings` 不同步 | 不在 `_collections` 內 → 顯示名稱、目標體重、熱量目標等**不會離開裝置** |
| 身分是匿名 uid | `FirebaseBootstrap.tryInit()` → `signInAnonymously()` |
| 無分析/廣告/當機回報 SDK | `pubspec.yaml` 只有 `firebase_core` / `firebase_auth` / `cloud_firestore` |
| 傳輸加密 | Firestore SDK 一律走 HTTPS/TLS |
| App 內可刪除 | 設定 → 資料與隱私(三種範圍) |
| 無設定檔時不上傳任何東西 | 缺 `google-services.json` → 初始化失敗 → 純本機模式 |

**關鍵觀念:** Play 定義的「collected」是<u>資料離開裝置</u>。只存在本機的東西不算收集。
所以「顯示名稱」即使使用者填真名,也**不申報**(但只要哪天把 settings 加進同步,就得改申報 Name)。

## 第一部分:資料安全性總問

| 問題 | 答案 | 說明 |
|---|---|---|
| 你的 App 是否收集或分享必填的使用者資料類型? | **是** | 有雲端同步版本 |
| 收集的資料是否全程加密傳輸? | **是** | Firestore 一律 TLS |
| 是否提供使用者要求刪除資料的方式? | **是** | 並填入刪除請求網址(見下) |
| 資料刪除網址 | `https://<你的 Pages 網域>/delete-account.html` | 必須公開可存取、免登入 |

## 第二部分:資料類型逐項

### ✅ 要申報

**1. Personal info → User IDs(使用者 ID)**
- 收集:是｜分享:否｜處理方式:非暫時性儲存
- 是否必填:**必填**(匿名帳號在啟動時自動建立,使用者無法事前選擇)
- 用途:**App functionality**(僅用於把雲端資料歸戶)
- 理由:Firebase 匿名 uid 隨資料一起存在雲端

**2. Health and fitness → Health info(健康資訊)**
- 收集:是｜分享:否｜必填:必填｜用途:App functionality
- 涵蓋:體重、體脂率、飲食熱量與營養素

**3. Health and fitness → Fitness info(健身資訊)**
- 收集:是｜分享:否｜必填:必填｜用途:App functionality
- 涵蓋:訓練紀錄(動作、組數、次數、重量、RPE、時間)、課表、自訂動作

> **「必填 vs 可選」的取捨:** 目前同步預設開啟、帳號自動建立,所以誠實答案是「必填」。
> 若日後改成首次啟動詢問是否啟用雲端,就能改成「使用者可選擇」。

### ❌ 不申報(且要能說明為什麼)

| 類型 | 為什麼不申報 |
|---|---|
| Name / Email / Phone | 完全不收集,App 沒有註冊或登入流程 |
| 顯示名稱 | 只存在本機 `AppSettings`,不在同步集合內 → 不算「collected」 |
| Location | 無權限、無程式碼 |
| Photos / Contacts / Calendar / SMS | 無權限、無程式碼 |
| App activity / App performance | 無分析、無當機回報 SDK |
| Device or other IDs | 未自行收集廣告 ID 或裝置 ID |
| Financial info | 無付費功能 |

### ⚠️ 需要你自己判斷的一項

Firebase Authentication 與 Firestore 在運作時會看到 **IP 位址**(Google 用於防濫用與安全)。
Play 對「僅為安全用途、且不留存」的資料有豁免空間,但認定權在 Google。

- 保守作法(建議):維持不申報,但在隱私權政策中已寫明使用 Firebase 及其條款連結 —— 目前已如此。
- 若審查方提出疑義:改申報 **Device or other IDs → 收集/不分享/App functionality + Fraud prevention, security**。

**若日後加入 Crashlytics 或 Analytics(RELEASE.md 的待辦),申報內容一定會變多**:
至少要加 App info and performance → Crash logs、Device or other IDs,並檢視 Analytics 的預設收集項目。

## 第三部分:其他相關表單

| 項目 | 位置 | 現況 |
|---|---|---|
| 隱私權政策網址 | Play Console → App content → Privacy policy | `https://<你的 Pages 網域>/`(docs/legal/index.html) |
| 資料刪除網址 | Data safety 表單內 | `.../delete-account.html` |
| 廣告聲明 | App content → Ads | **無廣告** |
| 目標對象與內容 | App content → Target audience | 選 18 歲以上或 13+;**不要**勾選兒童相關 |
| 健康 App 聲明 | App content(若出現) | 本 App **未**使用 Health Connect,也不宣稱醫療用途 |
| 內容分級問卷 | App content → Content rating | 健身工具,無敏感內容 |

## 待填欄位清單(程式碼給不出來的)

- [ ] 開發者名稱(需與 Play 開發者帳號一致)— `docs/legal/*.html` 兩處
- [ ] 聯絡信箱 — `docs/legal/*.html` 三處
- [ ] Firestore 專案區域 — `docs/legal/index.html` 第三節
- [ ] 兩個頁面的實際網址 — 填回本檔與 Play Console
