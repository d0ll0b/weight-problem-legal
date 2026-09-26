# 資料庫設計分析 & Firebase 串接指南

> 對應 DEVELOPMENT_PLAN.md,分析日期 2026-07-13,現況 v1.1.0+2(Phase 0~2 完成)。

---

## 一、資料表總覽

### 設計原則(已落地,`lib/data/db/tables.dart`)

所有表共用 `SyncColumns` mixin,這是整個資料層最重要的決定:

| 欄位 | 型別 | 用途 |
|------|------|------|
| `id` | TEXT (UUID) | 主鍵。**不用自增整數**:多裝置離線同時新增會撞號,UUID 天生無衝突 |
| `createdAt` | DATETIME | 建立時間 |
| `updatedAt` | DATETIME | **增量同步的游標**:「拉取比上次同步新的資料」全靠它 |
| `deletedAt` | DATETIME? | **soft delete(墓碑)**:真刪除無法同步到其他裝置,標記刪除才行 |

沒有這四個欄位,接任何雲端(Firebase 或 Supabase)都要重寫資料層 —— 這就是計畫「風險提醒 #2」的原因。

### 現有 8 張表(Phase 0~2,已實作)

```
exercises ──┬──< routine_exercises >── routines
            │
            └──< workout_sets >── workout_sessions ──(nullable)── routines
body_weights                （>──< 代表一對多)
diet_entries
app_settings (key-value,不同步)
```

**1. `exercises` 動作庫**

| 欄位 | 說明 |
|------|------|
| name | 動作名稱 |
| muscleGroup | chest/back/legs/shoulders/arms/core/cardio |
| equipment | barbell/dumbbell/machine/cable/bodyweight/other |
| description, mediaUrl | 說明文字、教學影片/圖(1-4) |
| isCustom | false = 內建種子(97 個,id 為 `sd-*`)。**內建動作不需同步**,各裝置種子一致 |

**2. `routines` 課表** — name, note, folder(資料夾分組), position(排序), sourceUserId(Phase 4 複製來源)

**3. `routine_exercises` 課表明細** — routineId, exerciseId, position, targetSets, targetReps, **restSeconds(每動作個別休息秒數,體驗關鍵)**

**4. `workout_sessions` 訓練** — routineId?(空白訓練為 null), startedAt, **endedAt(null = 進行中,App 閃退重開靠它恢復)**, note

**5. `workout_sets` 訓練組** — sessionId, exerciseId, setIndex, weight, reps, isWarmup, rpe?, **completedAt(null = 未打勾;統計只計已完成的正式組)**

**6. `body_weights` 體重** — date(日粒度,一天一筆 upsert), weight, bodyFat?

**7. `diet_entries` 飲食** — date(日粒度), mealType(breakfast/lunch/dinner/snack), name, calories, protein?, carbs?, fat?

**8. `app_settings`** — 純 key-value(default_rest_seconds、daily_calorie_goal、seed_version…)。裝置偏好,**刻意不進同步範圍**

### 查詢熱點(決定了為什麼選 SQL)

- 統計頁:`sets × sessions` join + 週聚合(Volume/Reps)、每日 max(Best/1RM)—— 純 SQL 一句話
- 預填上次數值:「該動作最近一次 session 的各組」
- PR 判定:「該動作歷史最大重量(排除本次)」
- 這些在關聯式資料庫是自然查詢,在 NoSQL 要嘛全量下載本地算、要嘛預先反正規化 —— **這是後面 Firebase 架構建議的核心依據**

### 未來各 Phase 需要新增的表

| Phase | 表 | 關鍵欄位 | 備註 |
|-------|-----|---------|------|
| P3 同步 | `sync_state`(本地) | lastPulledAt, lastPushedAt(每張表一筆) | 增量同步游標 |
| P4 社群 | `profiles` | userId, displayName, avatarUrl, bio, isPublic | 雲端為主 |
| P4 | `follows` | followerId, followeeId | 純雲端 |
| P4 | `feed_posts` | userId, sessionSummary(冗餘快照), likeCount | 動態牆用**快照**而非 join,別人刪課表不影響貼文 |
| P4 | `post_likes` / `post_comments` | postId, userId, content | 純雲端 |
| P5 穿戴 | `heart_rate_samples` | sessionId, at, bpm | 高頻資料,只存本地 + 健康平台,**不上雲**(量大無social價值) |
| P6 AI | `ai_generations` | userId, prompt 摘要, resultRoutineId, createdAt | 配額控制與除錯用 |

---

## 二、Firebase 串接

### 先說結論(20 年經驗的誠實建議)

原計畫 Phase 3 選的是 **Supabase(Postgres)**,理由:訓練資料高度關聯 + 統計是 SQL 天生強項 + RLS 規則簡單。**這個理由今天仍然成立。**

但 Firebase 也完全可行,而且有它贏的地方:Auth 最成熟、Crashlytics/Analytics/FCM 一站式、免費額度對個人 App 很夠。**可行的前提是一個架構鐵律:**

> **本地 Drift 永遠是唯一資料真相(source of truth),Firestore 只當「同步與備份層」。
> 所有統計、查詢、畫面都讀本地 SQL,永遠不直接查 Firestore 算統計。**

理由:Firestore 按「文件讀取次數」計費且沒有 SQL 聚合 —— 在雲端算週 Volume 要把每組都讀一遍,又慢又燒錢;在本地 SQL 算是 0 成本毫秒級。你的 App 是 offline-first,這個架構順理成章。

### 兩邊比較(給你做最終決定)

| | Firebase (Firestore) | Supabase (Postgres) |
|---|---|---|
| 資料模型 | 文件/子集合,要反正規化 | 關聯式,**本地 schema 直接鏡射** |
| 同步查詢「updatedAt > 游標」 | ✅(要建索引) | ✅ |
| 伺服器端統計 | ❌ 沒有聚合,靠 Cloud Functions | ✅ SQL/View 直接算 |
| 權限 | Security Rules(路徑式,簡單場景很簡單) | RLS(SQL 式,複雜場景更有力) |
| 社群(follow/動態牆) | 要 fan-out + Functions,工程較繞 | join 直接查 |
| Auth | 業界最成熟,匿名→正式帳號連結內建 | 夠用 |
| Crashlytics/Analytics/FCM | ✅ 一站式 | 要另接(Sentry 等) |
| 免費額度 | 讀 5 萬/日、寫 2 萬/日、1GiB | 500MB DB、5 萬月活躍 Auth |

**混搭也合理**:同步用 Supabase,但 Crashlytics + Analytics + FCM 照用 Firebase(計畫本來就列了 Firebase Analytics)。兩者不衝突。

### Firestore 資料模型(若選 Firebase)

關聯式 → 文件式的對映,原則:**會一起讀寫的就內嵌,需要獨立分頁查詢的才開子集合**。

```
users/{uid}                          profile + 同步中繼資料
users/{uid}/exercises/{id}           只存 isCustom=true(內建 97 個不同步)
users/{uid}/routines/{id}            課表文件,routine_exercises 內嵌成陣列 ↓
    { name, folder, position, updatedAt, deletedAt,
      exercises: [ {exerciseId, position, targetSets, targetReps, restSeconds}, ... ] }
users/{uid}/sessions/{id}            訓練文件,workout_sets 內嵌成陣列 ↓
    { routineId, startedAt, endedAt, updatedAt, deletedAt,
      sets: [ {exerciseId, setIndex, weight, reps, isWarmup, rpe, completedAt}, ... ] }
users/{uid}/bodyWeights/{id}
users/{uid}/dietEntries/{id}
```

為什麼 sets 內嵌而不是子集合:一次訓練頂多幾十組,遠低於 1MB 文件上限;拉一次訓練 = **1 次文件讀取**而不是 N 次(省 30 倍讀取費);伺服器端從不需要跨 session 查單組(統計在本地)。`routine_exercises` 同理。

**進行中的訓練(endedAt=null)不推雲端**,結束才推 —— 避免每打一勾寫一次雲(寫入費 + 衝突面)。

### 同步引擎設計(對應計畫 3-3,LWW)

本地加一張 `sync_state` 表(tableName, lastPulledAt, lastPushedAt)。

**推(push)**:每張表撈 `updatedAt > lastPushedAt` 的列 → 組成文件 batch 寫入(≤500/batch)→ 成功後推進 lastPushedAt。
**拉(pull)**:每個集合查 `where updatedAt > lastPulledAt orderBy updatedAt`(需要索引)→ 逐筆與本地比:**雲端 updatedAt > 本地 updatedAt 才覆蓋**(last-write-wins)→ 推進 lastPulledAt。
**刪除**:`deletedAt` 墓碑照常同步,永不真刪文件。
**觸發時機**:App 啟動、結束訓練後、進背景前;失敗靜默重試,UI 永不等待同步。

時鐘偏移註記:純 LWW 靠裝置時鐘,個人 App 兩台裝置場景足夠;要更嚴謹可在寫入時用 `FieldValue.serverTimestamp()` 存一個伺服器時間欄位做仲裁。

### Security Rules(第一版就這幾行)

```
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {
    match /users/{uid}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
  }
}
```

Phase 4 公開課表/動態牆再開 `public_routines`、`posts` 等頂層集合與對應規則。

### 實際串接步驟(對應計畫 Phase 3 的 3-1 ~ 3-4)

1. **Firebase Console 建專案** → 加 Android App(套件名 `com.weightproblem.weight_problem`,下載 google-services.json)與 iOS App(需要 Mac 那步再補)。啟用 Authentication(Anonymous + Google + **Sign in with Apple,iOS 上架硬性要求**)與 Firestore。
2. **接 SDK**:
   ```
   dart pub global activate flutterfire_cli
   flutterfire configure          # 自動產 firebase_options.dart 與平台設定
   flutter pub add firebase_core firebase_auth cloud_firestore
   ```
   `main()` 裡 `await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform)`。
3. **匿名帳號起步**:首次啟動 `signInAnonymously()`,使用者無感;之後登入 Google/Apple 時用 `linkWithCredential()` **把匿名帳號升級**,本地與雲端資料原地保留 —— 這正是計畫 3-2「匿名可繼續用,登入後合併」的 Firebase 標準解法。
4. **寫同步引擎**(上節設計,約一週,計畫也預留一週)。`lib/data/sync/` 新增 `sync_service.dart` + `sync_state` 表。
5. **順手接 Crashlytics + Analytics**(取代原計畫的 Sentry,少一個帳號):`flutter pub add firebase_crashlytics firebase_analytics`,release 模式掛 `FlutterError.onError`。
6. **多裝置驗證**:兩台裝置(或裝置+模擬器)交叉改課表/補訓練,驗證收斂與墓碑傳播 → 發 v1.2。

### 費用直覺(免費 Spark 方案)

個人訓練 App 一個活躍用戶:每天 1 次訓練 ≈ 幾次文件寫入 + 同步拉取幾次讀取,一天 < 50 次操作。免費額度(讀 5 萬/日)支撐**上千日活**沒問題;真正燒讀取的是動態牆(Phase 4),到時再優化。

---

## 三、行動建議

- 同步後端**二選一**:願意用兩個服務 → 依原計畫 Supabase(schema 鏡射最省工);想單一供應商全包 → Firebase,照上面架構做,鐵律是統計永遠在本地算。
- 不論選哪個,**Crashlytics/Analytics 現在就可以接**(發布準備的一部分,不依賴同步)。
- 資料表不需要為 Firebase 改任何東西 —— `SyncColumns` 第一天就把路鋪好了。
