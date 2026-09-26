# 法務頁面(隱私權政策 / 資料刪除)

兩個靜態頁,沒有任何外部資源,丟到任何靜態主機都能用:

- `index.html` — 隱私權政策(Play Console → App content → Privacy policy)
- `delete-account.html` — 帳號與資料刪除說明(Data safety 表單的「資料刪除網址」)

內容對照見 [`../PLAY_DATA_SAFETY.md`](../PLAY_DATA_SAFETY.md)。

## 上線前必須填掉的欄位

檔案裡用 `[待填:...]` 標出來了,搜尋「待填」就能全部找到:

- 開發者名稱(要與 Google Play 開發者帳號顯示名稱一致)
- 聯絡信箱(建議用專門收這類請求的信箱,不要用日常私人信箱)
- Firestore 專案所在區域

## 託管:為什麼不能直接用這個 repo

`d0ll0b/weight-problem-app` 目前是**私有** repo,而私有 repo 的 GitHub Pages 需要付費方案。
Play Console 要求隱私權政策網址「公開、免登入、可直接存取」,所以建議:

**開一個公開的小 repo 專門放這兩頁**

```
gh repo create weight-problem-legal --public
```

把 `index.html` 與 `delete-account.html` 放進去 → Settings → Pages →
Source 選 `main` / `/ (root)` → 得到:

- `https://d0ll0b.github.io/weight-problem-legal/`
- `https://d0ll0b.github.io/weight-problem-legal/delete-account.html`

其他等效選項:Cloudflare Pages、Netlify(都有免費方案),或把主 repo 轉公開後
用 `/docs` 當 Pages 來源。

## 上線後自我檢查

1. 用無痕視窗開兩個網址,確認不需登入就看得到
2. 確認頁面裡沒有殘留「待填」字樣
3. 手機上開一次(兩頁都做了窄螢幕排版)
4. 網址填進 Play Console 後,政策內容有變更就要更新頁面上的「最後更新」日期
