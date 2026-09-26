# 啟用 Firebase 雲端同步(Phase 3)

> 程式側已完成(2026-07);雲端同步預設**停用**,App 完全離線可用。
> 這份是「要開啟同步時」的操作步驟。啟用前不需要做任何事。

## 架構回顧

- 本地 Drift 是**唯一資料真相**,Firestore 只是同步/備份層。
- 所有統計、查詢、畫面都讀本地 SQL,永遠不查 Firestore 算東西(省讀取費、快)。
- 同步引擎:`updatedAt` 微秒游標增量同步,衝突以最後寫入為準(LWW),
  刪除靠墓碑(`deletedAt`)傳播;進行中訓練不推、內建動作不同步。
- 程式碼:`lib/data/sync/`(SyncService / SyncBackend / FirestoreSyncBackend)、
  `lib/core/firebase/firebase_bootstrap.dart`。

## 一次性設定

1. **建立 Firebase 專案**:<https://console.firebase.google.com> → 新增專案。
2. **啟用服務**:
   - Authentication → 開啟 **匿名**(必要)、之後要帳號合併再加 Google /
     Sign in with Apple(iOS 上架必要)。
   - Firestore Database → 建立(正式模式)。
3. **接上 App**(在專案根目錄):
   ```bash
   dart pub global activate flutterfire_cli
   flutterfire configure
   ```
   選剛建立的專案與 android/ios 平台。這會產生 `lib/firebase_options.dart`,
   並把 `google-services.json` / `GoogleService-Info.plist` 放到對的位置。
4. **改一行 main.dart**:把 `firebase_bootstrap.dart` 裡的
   `Firebase.initializeApp()` 換成帶 options 的版本:
   ```dart
   import '../../firebase_options.dart';
   await Firebase.initializeApp(
     options: DefaultFirebaseOptions.currentPlatform,
   );
   ```
5. **部署安全規則**:把 `firebase/firestore.rules` 貼到
   Console → Firestore → 規則,或用 CLI:
   ```bash
   firebase deploy --only firestore:rules
   ```

完成後 App 啟動會自動用匿名帳號登入並在背景同步;設定頁「雲端同步」
會顯示帳號與上次同步時間,可手動觸發。

## 驗證多裝置(計畫 3-4)

兩台裝置(或裝置 + 模擬器)裝同一版 App,各自：新增訓練 / 改課表 /
記體重 → 到設定頁按同步 → 另一台按同步 → 確認資料收斂、刪除也會傳播。

## 之後(帳號合併)

匿名帳號可無縫升級:使用者登入 Google / Apple 時用
`linkWithCredential()`,本地與雲端資料原地保留(計畫 3-2)。這段 UI
等要做正式登入頁時再補;目前匿名帳號already 能跨裝置同步(靠同一 uid)。

## 費用

個人使用一天 < 50 次讀寫,免費額度(讀 5 萬/日、寫 2 萬/日)可撐上千日活。
真正燒讀取的是 Phase 4 社群動態牆,到時再優化。
