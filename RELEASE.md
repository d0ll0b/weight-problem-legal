# 發布手冊(1-19 / 2-6)

> 程式側的發布準備已完成;這份文件是「環境備妥後」的操作步驟。
> 最後更新:2026-08-21,對應版本 1.1.0+2。

## 已完成(不用再做)

- ✅ App icon:來源圖由 `tool/generate_launcher_assets_test.dart` 用 Canvas 繪製,
  各平台尺寸由 `flutter_launcher_icons` 生成(Android 自適應 icon、iOS、Web favicon 都有)
- ✅ 啟動畫面:`flutter_native_splash`,深色底 `#0E0F13` + 置中槓鈴(含 Android 12+ 專用規格)
- ✅ 版本號:`pubspec.yaml` → `1.1.0+2`(`版本名+build 號`,每次上傳商店 build 號 +1)
- ✅ Android release 簽名骨架:`android/app/build.gradle.kts` 會讀 `android/key.properties`,
  沒有該檔時自動退回 debug 簽名(開發不受影響)
- ✅ App 顯示名稱「體重是個問題」、minSdk 26、iOS 15、通知權限宣告

### 重新生成 icon / 啟動畫面(改了設計才需要)

```
flutter test tool/generate_launcher_assets_test.dart   # 重畫來源 PNG
dart run flutter_launcher_icons                        # 出各平台 icon
dart run flutter_native_splash:create                  # 出啟動畫面
```

## Android:出第一個 APK(自己裝手機測試)

> **測試不需要 keystore。** `android/app/build.gradle.kts` 在找不到 `key.properties`
> 時會退回 debug 簽名,`flutter build apk --release` 照樣出得來、照樣裝得上手機。
> keystore 只有要上架 Google Play 才需要(見下一節)。

### 前置:兩個環境條件

1. **JDK 17 以上。** 本機目前 `JAVA_HOME` 指向 **JDK 8**(`C:\Program Files\Java\jdk1.8.0_152`),
   而本專案的 Gradle 9.1 + AGP 需要 17+,直接打包會失敗。
   裝 Android Studio 會一併帶 JetBrains Runtime(JDK 21),Flutter 會優先採用它,
   這個問題就自動解決 —— 這也是建議走 Android Studio 而非只裝 cmdline-tools 的主因。
   若堅持不裝 Studio,自己裝一份:
   ```
   winget install EclipseAdoptium.Temurin.17.JDK
   flutter config --jdk-dir "C:\Program Files\Eclipse Adoptium\jdk-17..."
   ```
2. **Android SDK**(目前未安裝):
   ```
   winget install Google.AndroidStudio
   ```
   首次啟動精靈會裝 SDK / platform-tools / build-tools。裝完:
   ```
   flutter doctor --android-licenses     # 全部 y
   flutter doctor                        # Android toolchain 應該變成 [√]
   ```

### 打包與安裝

```
flutter build apk --release
```
產物在 `build\app\outputs\flutter-apk\app-release.apk`。裝進手機任選一種:

- **USB**:手機開「開發人員選項 → USB 偵錯」,接上後 `flutter install --release`
  (或 `adb install -r build\app\outputs\flutter-apk\app-release.apk`)
- **不接線**:把 apk 丟雲端硬碟/傳到手機,點開安裝(要允許「安裝不明來源應用程式」)

想同時支援舊機並縮小體積可加 `--split-per-abi`,會分別產出 armeabi-v7a / arm64-v8a APK。

### 實機重點測試(web 上測不到的)

休息計時器背景通知、震動/音效、通知權限請求(Android 13+)、
App 被系統回收後進行中訓練的恢復、深色/淺色系統主題切換。

## Firebase 設定檔:release 版的硬性條件

`android/app/google-services.json` 在 `.gitignore` 內,但**正式版不能少**:

- 有這個檔 → `app/build.gradle.kts` 才會套用 `com.google.gms.google-services` plugin,
  `Firebase.initializeApp()` 才讀得到設定,雲端同步才會活。
- 沒有這個檔 → App 靜默降級成純本機模式。使用者仍看得到「雲端同步」那一列,
  點了永遠沒反應 —— 這種包上架就是事故。

所以 release 打包時 `preReleaseBuild` 會直接擋下來並說明原因。
真的要出一個純本機版:

```
flutter build appbundle --release -PallowMissingFirebase=true
```

iOS 對應的檔案是 `ios/Runner/GoogleService-Info.plist`(目前同樣未納管)。

## Google Play 內部測試(要正式簽名時才做)

1. **產生正式 keystore(一次性,務必備份,遺失後無法更新已上架的 App)**:
   ```
   keytool -genkey -v -keystore %USERPROFILE%\weight-problem-release.jks ^
     -keyalg RSA -keysize 2048 -validity 10000 -alias weightproblem
   ```
2. 複製 `android/key.properties.example` → `android/key.properties`,填入密碼與路徑。
3. `flutter build appbundle --release` → `build\app\outputs\bundle\release\app-release.aab`
4. Play Console 建 App(內部測試軌道)→ 上傳 .aab → 拉 5~10 位測試者。
   需要一次性 25 USD 開發者帳號。

## 最快的臨時測試:web 版跑在手機瀏覽器

不用裝任何東西,適合純 UI / 流程驗證(**測不到通知、震動、背景行為**):

```
flutter build web --release
```
把 `build\web` 用任意靜態伺服器分享到區網,手機連同一個 Wi-Fi 開
`http://<電腦區網IP>:8000` 即可。資料存在手機瀏覽器的 IndexedDB,是獨立的一份。

## iOS:需要一台 Mac

1. Mac 上裝 Xcode + `flutter doctor` 過關;Apple Developer Program(99 USD/年)。
2. `open ios/Runner.xcworkspace` → Signing & Capabilities 選 Team,
   Bundle ID 用 `com.weightproblem.weightProblem`(或到 Apple Developer 重新登記)。
3. `flutter build ipa --release` → Xcode Organizer 或 Transporter 上傳。
4. App Store Connect → TestFlight → 內部測試者。

## 資料刪除(已完成程式側)

設定頁「資料與隱私」區提供三條路徑,對應 Google Play 的使用者資料政策:

- 清除這台裝置上的資料
- 刪除雲端資料(清空 Firestore 並關閉同步,本機保留)
- 刪除帳號與所有資料(雲端文件 → 匿名帳號 → 本機資料,順序不可對調:
  帳號一刪就失去認證身分,Security Rules 會擋住後續寫入,雲端文件會變孤兒)

刪除後會寫入 `cloud_sync_opt_out` 旗標,啟動時不再自動開新的匿名帳號。

網頁版的刪除說明與隱私權政策在 [`legal/`](legal/),Data safety 表單的逐項答案見
[`PLAY_DATA_SAFETY.md`](PLAY_DATA_SAFETY.md)。兩頁都還有 `[待填:...]` 欄位要補
(開發者名稱、聯絡信箱、Firestore 區域),並且要先託管到公開網址才能填進 Play Console
—— 主 repo 是私有的,託管方式見 `legal/README.md`。

## 上架前待辦(下一輪可做)

- [ ] Sentry / Crashlytics 崩潰回報:註冊拿 DSN 後接 `sentry_flutter`
      (刻意等有 DSN 才加,避免帶著沒用的依賴)
- [x] 隱私政策頁與 Data safety 對照表(`legal/`、`PLAY_DATA_SAFETY.md`)——
      **還要填入待填欄位並託管到公開網址**
- [ ] 把兩個網址填進 Play Console(App content → Privacy policy、Data safety → 刪除網址)
- [ ] 商店素材:截圖(手機各尺寸)、簡介文案、分級問卷
