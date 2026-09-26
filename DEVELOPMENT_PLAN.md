# 「體重是個問題」健康管理 App 開發計畫

> Flutter (iOS / Android) · 建立日期:2026-07-07

---

## 一、資深工程師的第一個建議:重新分層 MVP

你列的 13 項功能全部做完才上線,至少要 6~9 個月,風險太高。
真正的 MVP 應該是「一個人可以完整記錄一次訓練並看到進步」,其他都是成長功能。

| 層級 | 功能 | 理由 |
|------|------|------|
| **P0 — 真 MVP(先上架)** | 動作庫、課表規劃、訓練紀錄、休息計時器、歷史紀錄、Dashboard、體重紀錄 | 單機可用、不需後端、核心價值閉環 |
| **P1 — 第二版** | 統計圖表(Volume / Best / 1RM)、飲食記錄、帳號與雲端同步 | 留存功能,需要累積資料後才有價值 |
| **P2 — 第三版** | 社群(follow / 複製課表 / 分享)、Apple Health / Health Connect | 需要後端與用戶基數 |
| **P3 — 第四版** | Apple Watch / Wear OS、AI 課表與建議 | 開發成本最高,且依賴前面所有資料 |

P0 採 **offline-first(本地資料庫優先)**:不用註冊就能用,之後再把同步補上。這是 Strong、Hevy 等成功健身 App 的共同路線。

---

## 二、技術選型

| 項目 | 選擇 | 理由 |
|------|------|------|
| 框架 | Flutter 3.x + Dart 3 | 雙平台單一程式碼 |
| 狀態管理 | Riverpod 2 (+ riverpod_generator) | 可測試、編譯期安全 |
| 本地資料庫 | Drift (SQLite) | 訓練資料是高度關聯式(課表→動作→組數),SQL 天生適合統計查詢 |
| 路由 | go_router | 官方維護、支援 deep link(社群分享會用到) |
| 後端 (P1 起) | Supabase | Postgres 關聯式資料 + Auth + Storage + Edge Functions,社群功能好做,免費額度夠 |
| 圖表 | fl_chart | 免費、客製化能力夠 |
| 影片 | video_player + chewie;影片放 CDN(Cloudflare R2/Stream),MVP 期可先用 YouTube 嵌入或 GIF 示意圖 | 自製影片成本高,先用替代方案 |
| 本地通知 | flutter_local_notifications | 休息計時器背景到時提醒(iOS 背景限制的解法) |
| 健康整合 | `health` package | 同一套 API 打 HealthKit 與 Health Connect |
| Watch | Apple Watch:原生 SwiftUI + WatchConnectivity;Wear OS:原生 Compose 或 Flutter(實驗性) | Flutter 不能寫 watchOS,這是原生工程 |
| AI | 後端 Edge Function 呼叫 Claude API | 金鑰不落地、可控成本 |
| 崩潰/分析 | Sentry + Firebase Analytics | 上線必備 |
| CI/CD | GitHub Actions + Fastlane | 自動打包上架 |

### 架構:Feature-first + 分層

```
lib/
├── core/            # 主題、路由、共用 widget、工具(1RM 計算等)
├── data/            # Drift 資料庫、DAO、(未來) Supabase repository
├── features/
│   ├── dashboard/
│   ├── exercises/     # 動作庫
│   ├── routines/      # 課表規劃
│   ├── workout/       # 進行中訓練 + 休息計時器
│   ├── history/       # 歷史紀錄、訓練日曆
│   ├── stats/         # 統計圖表
│   ├── body/          # 體重紀錄
│   ├── diet/          # 飲食記錄
│   ├── social/        # 社群
│   ├── ai/            # AI 建議
│   └── settings/
└── main.dart
```

每個 feature 內部:`data / domain / presentation` 三層。

### 核心資料模型(Drift)

```
exercises        id, name, muscle_group, equipment, media_url, is_custom
routines         id, name, note, folder, created_at, source_user_id(複製來源)
routine_exercises id, routine_id, exercise_id, order, target_sets,
                  target_reps, rest_seconds
workout_sessions id, routine_id?, started_at, ended_at, note
workout_sets     id, session_id, exercise_id, set_index, weight, reps,
                 is_warmup, rpe?, completed_at
body_weights     id, date, weight, body_fat?
diet_entries     id, date, meal_type, name, calories, protein, carbs, fat
settings         unit(kg/lb), default_rest_seconds, ...
```

這個 schema 從第一天就要設計好 `uuid` 主鍵與 `updated_at`,P1 做雲端同步時才不用大改。

---

## 三、逐步執行清單

> 每一步都小到可以在 1 次工作階段內完成,並附驗收標準。
> 建議節奏:一人全職約每天 2~4 步。

### Phase 0:專案骨架(約 1 週)

- [ ] **0-1 建立 Flutter 專案**:`flutter create`,設定 bundle id(如 `com.yourname.weightproblem`)、App 名稱「體重是個問題」、最低版本 iOS 15 / Android 8。驗收:兩平台模擬器都能跑起 demo。
- [ ] **0-2 Git + 目錄結構**:建 repo、`.gitignore`、按上面 feature-first 結構建空資料夾、加入 `analysis_options.yaml`(very_good_analysis)。
- [ ] **0-3 加入核心套件**:riverpod、go_router、drift、flutter_local_notifications、intl。驗收:`flutter pub get` 無錯、能編譯。
- [ ] **0-4 主題與設計系統**:定義 ColorScheme(深色為主,健身 App 慣例)、字體、spacing 常數、共用元件(主按鈕、卡片、輸入框)。驗收:一個 style gallery 頁面展示所有元件。
- [ ] **0-5 路由骨架**:go_router + 底部導覽列 4 個 tab(Dashboard / 課表 / 歷史 / 設定),每頁先放 placeholder。
- [ ] **0-6 Drift 資料庫**:建立上述 schema、migration 機制、寫第一個 DAO 單元測試。驗收:測試通過,App 啟動能開 DB。

### Phase 1:核心訓練閉環(約 4~5 週)——完成即可 TestFlight 內測

**動作庫**
- [ ] **1-1 內建動作種子資料**:整理 100~150 個常見動作(名稱、肌群、器材、動作說明文字),做成 JSON,首次啟動匯入 DB。
- [ ] **1-2 動作庫列表頁**:搜尋、依肌群/器材篩選、動作詳情頁(先放文字說明 + 示意圖)。
- [ ] **1-3 自訂動作**:新增/編輯/刪除自訂動作。
- [ ] **1-4 教學影片**:詳情頁嵌入影片播放(先用免費示範 GIF 或 YouTube 嵌入,自製影片列為後續內容工作)。

**課表規劃**
- [ ] **1-5 課表列表頁**:建立/改名/刪除/排序 routine,支援資料夾分組。
- [ ] **1-6 課表編輯頁**:從動作庫加入動作、拖曳排序、設定每個動作的目標組數/次數/**個別休息秒數**。
- [ ] **1-7 課表範本**:內建 3~5 套新手課表(如 5x5、PPL、上下肢分化)。

**訓練紀錄(整個 App 的心臟)**
- [ ] **1-8 開始訓練流程**:從課表啟動 or 空白訓練;建立 `workout_session`。
- [ ] **1-9 記錄組數 UI**:每動作列出組數列,輸入重量/次數、打勾完成;自動帶入「上次訓練同動作的重量次數」作為預設值。這是體驗關鍵。
- [ ] **1-10 訓練中管理**:中途加動作/換動作/加組/刪組;上滑 keyboard 快速輸入;訓練計時(總時長)。
- [ ] **1-11 休息計時器**:打勾完成一組 → 自動啟動該動作設定的 rest timer;倒數浮動條 + 可加減 15 秒;到時震動/音效;**App 進背景時用排程本地通知提醒**(iOS 沒有背景計時,這是業界標準解法)。
- [ ] **1-12 結束訓練**:總結頁(時長、總 Volume、破 PR 的動作)、儲存;意外關閉 App 能恢復進行中的訓練(session 落地 DB)。

**歷史與 Dashboard**
- [ ] **1-13 歷史列表**:按日期倒序的訓練卡片,點入看完整明細。
- [ ] **1-14 訓練日曆**:月曆熱點標記有訓練的日子(table_calendar 套件)。
- [ ] **1-15 動作歷史**:在動作詳情頁看該動作的歷次重量紀錄(記錄時「查看上次重量」的來源)。
- [ ] **1-16 體重紀錄**:每日輸入體重(App 名稱是體重是個問題,這個要在 P0!)、簡單趨勢折線圖。
- [ ] **1-17 Dashboard v1**:本週訓練次數、連續週數 streak、最近一次訓練摘要、體重趨勢小卡、快速開始按鈕。
- [ ] **1-18 設定頁**:單位 kg/lb、預設休息秒數、資料匯出(CSV)。
- [ ] **1-19 內測發布**:App icon、啟動畫面、Sentry 接入、TestFlight + Google Play 內測軌道發布給 5~10 位朋友。

### Phase 2:統計與飲食(約 2~3 週)

- [ ] **2-1 統計查詢層**:寫 SQL 聚合查詢(週/月 Volume、每動作 Best Weight、Total Reps)+ 單元測試。
- [ ] **2-2 1RM 計算**:Epley 公式 `1RM = weight × (1 + reps/30)`,取歷史最佳估算值。
- [ ] **2-3 圖表頁**:fl_chart 畫 4 種圖(Volume 趨勢、Best Weight、Total Reps、1RM 趨勢),支援按動作/肌群/時間範圍切換。
- [ ] **2-4 Dashboard v2**:嵌入週 Volume 圖與肌群分布。
- [ ] **2-5 飲食記錄 v1**:每餐快速記錄(名稱、熱量、蛋白質/碳水/脂肪)、每日總計 vs 目標、常用食物清單。**先不要做食物資料庫掃碼**,那是無底洞,v1 手動輸入 + 常用清單就夠。
- [ ] **2-6 發布 v1.1**。

### Phase 3:帳號與雲端同步(約 2~3 週)

- [ ] **3-1 Supabase 專案建置**:Postgres schema 鏡射本地 schema、Row Level Security 規則。
- [ ] **3-2 登入**:Sign in with Apple(iOS 上架必要)+ Google;匿名可繼續用,登入後合併本地資料。
- [ ] **3-3 同步引擎**:以 `updated_at` + soft delete 做增量同步;衝突以最後寫入為準。這是本階段最難的一步,預留一週。
- [ ] **3-4 多裝置驗證 + 發布 v1.2**。

### Phase 4:社群(約 3~4 週)

- [ ] **4-1 個人檔案**:頭像、名稱、簡介、公開/私人設定。
- [ ] **4-2 Follow 系統**:追蹤/被追蹤、追蹤者列表。
- [ ] **4-3 動態牆**:追蹤對象的訓練完成卡片(時長、Volume、PR),讚與留言。
- [ ] **4-4 複製朋友課表**:公開課表詳情頁 + 一鍵複製到自己的課表(記錄 `source_user_id`)。
- [ ] **4-5 圖片分享**:把訓練總結/統計圖表渲染成分享圖(RepaintBoundary 截圖),share_plus 分享到 IG/Line。
- [ ] **4-6 檢舉/封鎖 + 隱私政策更新**(上架審核會查)、發布 v2.0。

### Phase 5:健康整合與穿戴(約 4~6 週)

- [ ] **5-1 HealthKit / Health Connect**:用 `health` 套件,寫入訓練(workout)與體重,讀取體重與心率;權限請求 UX。
- [ ] **5-2 Apple Watch App(原生 SwiftUI)**:訓練中顯示當前動作/組數、打勾完成、休息倒數、即時心率;WatchConnectivity 與手機同步。這是獨立的原生子專案。
- [ ] **5-3 Wear OS**:先做最小版(訓練中顯示 + 心率),評估用戶比例再決定投入。
- [ ] **5-4 心率入紀錄**:訓練明細顯示心率曲線與平均/最高心率。發布 v2.5。

### Phase 6:AI(約 2~3 週)

- [ ] **6-1 AI 課表生成**:問卷(目標/經驗/天數/器材)→ Edge Function 呼叫 Claude API,**限制只能輸出動作庫中存在的動作**(給它動作清單),回傳 JSON 直接建成 routine。
- [ ] **6-2 訓練建議**:根據近期 Volume/1RM 趨勢與恢復狀況,每週產生一段建議(進步停滯提醒、加重建議、肌群不平衡)。
- [ ] **6-3 成本控制**:每用戶每日次數上限、快取。發布 v3.0。

---

## 四、時程總覽(一人全職)

| 階段 | 內容 | 時間 | 累計 |
|------|------|------|------|
| Phase 0 | 骨架 | 1 週 | 1 週 |
| Phase 1 | 核心訓練閉環 → **內測** | 4~5 週 | ~6 週 |
| Phase 2 | 統計+飲食 → **正式上架** | 2~3 週 | ~9 週 |
| Phase 3 | 帳號同步 | 2~3 週 | ~12 週 |
| Phase 4 | 社群 | 3~4 週 | ~16 週 |
| Phase 5 | 健康+穿戴 | 4~6 週 | ~22 週 |
| Phase 6 | AI | 2~3 週 | ~25 週 |

## 五、風險提醒(20 年的坑)

1. **iOS 背景計時器**:App 進背景後 timer 會停,務必用「排程本地通知」實作休息提醒,不要嘗試 hack 背景執行。
2. **同步引擎**:比想像難 3 倍。第一天就用 UUID 主鍵 + `updated_at` + soft delete,否則 Phase 3 會重寫資料層。
3. **動作影片**:自製影片是內容工程不是程式工程,MVP 用示意圖/授權 GIF 頂著,別讓它擋住上線。
4. **食物資料庫**:掃碼與台灣食品資料庫是無底洞,v1 手動輸入即可。
5. **Watch**:Flutter 寫不了 watchOS,需要 Swift 能力,時程要獨立估。
6. **上架審核**:Sign in with Apple(有第三方登入就必須有)、隱私政策、健康資料使用聲明(HealthKit 審核特別嚴)、社群功能必須有檢舉/封鎖。
