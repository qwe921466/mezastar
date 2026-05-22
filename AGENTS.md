# AGENTS.md

## 專案概覽

這個專案是 `Mezastar 記帳本`，一個行動優先的單頁 PWA 工具，用來記錄 Mezastar 機台遊玩花費、玩家、地點、取得卡片、圖鑑收藏與簡易統計。

專案沒有打包流程、沒有 npm 依賴，也沒有後端服務。主要功能集中在 `index.html`，資料存在使用者瀏覽器的 `localStorage`，備份與還原透過 JSON 檔案完成。

## 檔案結構

- `index.html`：主要應用程式，包含 HTML、CSS、UI 結構與所有 JavaScript 邏輯。
- `manifest.json`：PWA 設定，包含應用名稱、啟動 URL、顏色與 icon 設定。
- `cards.json`：外部卡號資料表，啟動時由 `index.html` 嘗試載入並合併進內建卡片資料庫。
- `pika.png`：PWA icon 與網站 icon。
- `qrcode.jpg`：姊姊的玩家登入 QR Code 圖片。
- `qrcode-younger.jpg`：妹妹的玩家登入 QR Code 圖片。

## 技術與執行方式

此專案直接用瀏覽器開啟即可：

```text
file:///.../index.html
```

若需要驗證 `cards.json` 的 `fetch()` 載入流程，建議用本機靜態伺服器開啟：

```bash
python3 -m http.server 8080
```

再進入：

```text
http://127.0.0.1:8080/index.html
```

注意：直接用 `file://` 開啟時，部分瀏覽器可能會擋下 `fetch('./cards.json')`。程式已設計 fallback，會繼續使用 `index.html` 內建的 `CardDatabase`。

## UI 架構

`index.html` 內的主畫面分為五個 tab：

- `tab-play`：遊玩中心，包含 QR Code、定位、快速記帳與近期動態。
- `tab-pokedex`：圖鑑，顯示已登錄卡片，可依玩家過濾。
- `tab-history`：歷史紀錄，顯示完整花費明細並提供編輯/刪除。
- `tab-stats`：統計，顯示總花費、估計遊玩道數、姊姊/妹妹花費與出卡率。
- `tab-map`：據點，內嵌 Google My Maps 並連到官方查詢。

底部導覽列是絕對定位在 app shell 底部。外層 `.app-shell` 使用固定視窗高度，`main` 使用 `flex-1 min-h-0 overflow-y-auto`，讓中間內容區滾動，避免歷史紀錄太多時把底部導覽擠到頁面最下方。

## 樣式與設計慣例

- 使用 Tailwind CDN，不使用建置工具。
- 使用 Lucide CDN 提供 icon，呼叫 `重整圖示()` 會重新建立 lucide icon 並渲染自訂寶貝球 SVG。
- 主體是手機寬度體驗：外層容器 `max-w-md`，視覺上像一個固定寬度的手機 app。
- 新增 UI 時優先沿用現有的紅色主題、粗體字重、圓角卡片、邊框與陰影語言。
- 若新增會動態塞入 `innerHTML` 的內容，必須先用 `轉義HTML()` 處理使用者輸入。
- 若新增圖片 URL 欄位，必須先用 `清理網址()` 驗證。

## 資料模型

主要資料存在：

```js
localStorage['mezastar_entries']
```

每筆紀錄大致結構如下：

```js
{
  id: 'uuid-or-fallback-id',
  date: 'YYYY-MM-DD',
  amount: 30,
  player: '姊姊' | '妹妹' | '一起',
  location: '地點文字',
  note: '備註',
  cardName: '卡片名稱',
  cardId: '1-1-001',
  stars: '0' | '2' | '3' | '4' | '5' | '6' | 'sp',
  imageUrl: 'https://...',
  createdAt: 1710000000000
}
```

資料新增、匯入與啟動讀取都應經過：

- `正規化紀錄(item)`
- `正規化紀錄清單(items)`
- `存到本地()`

這些函式會處理欄位長度、有效玩家、有效星級、日期格式、圖片 URL、重複 ID 與無效金額。

## 主要 JavaScript 流程

- 啟動流程：`DOMContentLoaded`
  - `載入外部卡片資料()`
  - 綁定卡片預覽 input 事件
  - 從 `localStorage` 讀取並正規化資料
  - 初始化首頁快速記帳玩家為 `姊姊`
  - 呼叫 `更新畫面()`

- 新增快速紀錄：`首頁快速新增(amount)`
  - 產生一筆只有金額、玩家、日期與定位的紀錄。
  - 快速記帳只提供 `姊姊` 與 `妹妹`，不提供 `一起`。

- 玩家登入碼：`開啟QRCode()` 與 `切換QRCode(player)`
  - `QRCode資料` 管理姊姊與妹妹的 QR 圖片、顯示 ID 與按鈕色彩。
  - QR 放大彈窗會依目前首頁玩家預設開啟對應登入碼。

- 詳細新增/編輯：`開啟記帳彈窗(id)` 與 `儲存表單紀錄()`
  - `id` 存在時為編輯模式。
  - 沒有 `id` 時為新增模式，預設金額為 `30`。

- 渲染流程：`更新畫面()`
  - 排序紀錄。
  - 更新近期列表、圖鑑、歷史紀錄與統計。
  - 最後呼叫 `重整圖示()`。

- 備份還原：
  - `匯出資料()` 下載格式化 JSON。
  - `匯入資料(event)` 會先正規化資料，再讓使用者選擇合併或覆蓋。

## 卡片資料

`CardDatabase` 內建部分高星卡，包含 `name`、`stars` 與可選的 `dex`。若有 `dex`，自動帶入時會使用 PokeAPI 官方圖：

```text
https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/other/official-artwork/{dex}.png
```

`cards.json` 可以補充卡號與中文名稱：

```json
{
  "1-1-001": {
    "name_zh_tw": "超夢",
    "series": "1-1"
  }
}
```

若同一卡號同時存在於 `CardDatabase` 與 `cards.json`，外部資料會覆蓋或補足名稱與系列，但不會自動補星級與 dex。新增卡號時，若需要自動圖片與星級，仍要在 `CardDatabase` 補 `stars` 與 `dex`。

## 安全與穩定性注意事項

- 不要把使用者輸入直接插進 `innerHTML`。
- 動態文字要用 `轉義HTML()`。
- 動態圖片或連結 URL 要用 `清理網址()`。
- 修改資料結構時，要同步更新 `正規化紀錄()`、匯入流程與渲染邏輯。
- 不要移除 `main` 上的 `min-h-0 overflow-y-auto`，否則歷史紀錄過長時會再次把底部導覽推走。
- 不要把底部導航改成一般文件流元素，除非同時重新處理整個 app shell 的高度與捲動策略。
- 因資料只存在本機瀏覽器，任何清除網站資料或換裝置都可能遺失紀錄；備份功能是重要功能，不要輕易破壞。

## 驗證方式

至少執行 JavaScript 語法檢查：

```bash
node -e "const fs=require('fs'); const html=fs.readFileSync('index.html','utf8'); const m=html.match(/<script>([\\s\\S]*)<\\/script>/); new Function(m[1]); console.log('inline script syntax ok')"
```

若修改 `cards.json` 載入、地圖、PWA 或圖片相關功能，建議用本機伺服器驗證：

```bash
python3 -m http.server 8080
```

可用以下方式確認 `cards.json` 可被伺服器提供：

```bash
curl -s http://127.0.0.1:8080/cards.json
```

## 維護建議

- 這是一個小型單檔應用，優先保持簡單，不要引入框架或建置工具，除非需求已明顯超過單檔可維護範圍。
- 若功能變多，可優先拆分資料、渲染、表單與工具函式，但要先確認部署方式能支援多檔 JS。
- 若新增統計，請以 `紀錄清單` 的正規化資料為唯一來源。
- 統計卡片只顯示 `姊姊` 與 `妹妹` 的分項；總花費仍會計入所有紀錄，包含舊資料中的 `一起`。
- 若新增玩家、星級或欄位，請同步更新允許清單、表單選項、資料正規化、匯入還原與所有統計邏輯。
- 保持中文命名風格一致；既有程式大量使用中文函式與變數名稱，新增邏輯可沿用這個風格以降低理解成本。
