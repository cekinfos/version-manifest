# version-manifest

集中管理各 App 的「商店最新版本」資訊。App 端以一次 HTTP GET 取得本清單，比對本機 build
後決定是否提示使用者前往商店更新。

採用靜態檔案託管而非自建後端，理由：

- 雙平台（Android / iOS）共用同一份判斷來源，不需為 iOS 另外處理（Apple 無 in-app update API）
- 可做強制更新（更新一律強制，提示無法略過），這是 App Store 本身做不到的
- 發布時機完全自控，不受 iTunes Lookup API 的 CDN 延遲影響

## 檔案結構

```text
version-manifest/
├── .nojekyll           關閉 GitHub Pages 的 Jekyll 處理
├── README.md
└── cekapp/
    └── version.json    CekApp（志光雲）版本清單
```

## 發布網址

```text
https://cekinfos.github.io/version-manifest/cekapp/version.json
```

啟用步驟（整個 repo 只需做一次）：

1. GitHub repo → **Settings** → **Pages**
2. Source 選 **Deploy from a branch**
3. Branch 選 `main`、資料夾選 `/ (root)`，按 **Save**
4. 等 1–2 分鐘，用瀏覽器開上述網址，確認回傳的是 JSON 內容

**前提：repo 必須是 public。** 免費方案的 GitHub Pages 不支援 private repo；若本 repo 為 private，
需將其改為 public，或改用 Cloudflare Pages 等替代託管。

## version.json 欄位定義

| 欄位 | 型別 | 說明 |
|------|------|------|
| `schemaVersion` | int | 結構版本。App 端讀到不認識的版本時應直接放棄檢查（靜默），不可誤判 |
| `updatedAt` | string | 本檔最後更新時間（ISO 8601 UTC）。供人工確認 CDN 快取是否已更新 |
| `android` / `ios` | object | 分平台區塊，兩者結構相同 |
| `latestBuild` | int | 商店上架的最新 build，對應 csproj 的 `ApplicationVersion`。**唯一用於比對的欄位**，也是強制更新的唯一開關 |
| `latestVersion` | string | 對應 csproj 的 `ApplicationDisplayVersion`，僅供顯示，不參與比對 |
| `releaseNotes` | string | 更新說明。空字串代表不顯示說明區塊 |

## App 端契約

- 比對一律用整數：`latestBuild` vs `AppInfo.Current.BuildString`。**禁止字串比大小**
  （`"1.10.0" < "1.9.0"` 在字串比較下會得到錯誤結果）
- **更新一律強制**：只要 `latestBuild` 高於本機 build 即提示，且提示無法略過。
  本清單沒有「可略過的版本」這種狀態
- **App 啟動後只檢查一次**；檢查失敗（含網路不通）在本次執行期間不重試，須冷啟動才會再檢查
- 請求需加 cache-buster：`?t={unix timestamp}`，否則 CDN 與 HttpClient 快取會讓使用者拿到舊檔
- 取不到 / 解析失敗 / 逾時 → 一律視為「無更新」，**不得阻擋登入或任何既有流程**
- 商店網址寫死在 App 內，不從本檔取得（理由見下方鐵則 1）

## 更新流程

1. 商店新版本**審核通過且確定已上架後**，才更新對應平台區塊（不要提前更新）
2. 修改該平台的 `latestBuild`、`latestVersion`、`releaseNotes`，並同步更新 `updatedAt`
3. 驗證 JSON 合法：

   ```bash
   python -c "import json; json.load(open('cekapp/version.json'))" && echo OK
   ```

4. commit + push
5. 開啟發布網址確認內容已更新（必要時網址加 `?t=1` 繞過快取）

## 鐵則

1. **不放商店網址。** 靜態檔案沒有任何鑑別機制。若 DNS 遭劫持或 GitHub 帳號被盜，攻擊者可將
   使用者導向惡意安裝來源，而此時使用者正處於「App 叫我更新」的高信任狀態。商店網址一律寫死在
   App 內為常數。
2. **不放任何機密。** public repo 全世界可讀，不得出現 API key、內部端點或帳號資訊。
3. **`latestBuild` 誤植會立刻鎖死全體使用者。** 更新一律強制，且 `latestBuild` 是唯一開關，
   沒有第二道防線 —— 多打一位數就會讓所有人被要求更新到一個不存在的版本，且無法略過。
   每次修改務必逐字核對 csproj 的 `ApplicationVersion`。
4. **分平台各自維護。** iOS 審核比 Android 慢，共用一組版本號會讓 Android 使用者被提示一個
   iOS 尚未過審的版本。
5. **push 前必須驗證 JSON 合法。** 一個多餘的逗號會讓全體使用者的版本檢查失敗。
6. **快取約 10 分鐘。** GitHub Pages 固定回應 `Cache-Control: max-age=600`，push 後不會立即生效。

## 備註：CekApp 的三個發布通道

CekApp 有三個發布通道，各自的套件識別碼不同：

| 通道 | 套件 / bundle id | manifest 區塊 |
|------|------------------|---------------|
| iOS（App Store） | `com.cek.superbox` | `ios` |
| Android 手機（Google Play） | `com.cek.cekcloud` | `android` |
| Android TVBox（`Release-STB` 建置組態） | `com.cek.superboxapp` | **不適用** |

TVBox 版以 APK 檔案側載派發，更新由既有的 APK 派發流程處理，**不納入版本檢查**，因此本清單
不提供對應區塊。App 端在 `Release-STB` 建置中以 `#if STB` 完全停用版本檢查，不會讀取本清單。

本清單刻意不儲存套件識別碼（見鐵則 1），上表僅供維運人員對照使用。
