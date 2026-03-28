# Japan Receipt Tracker — Tech Stack

> 在日本旅行時，你怎麼記帳？每天手動輸入日文收據上的金額、品項、稅別，再自己翻譯分類？
>
> 我做了一個網頁，讓 AI 自動辨識日文收據、翻譯成繁體中文、分類記帳，在一個月的日本旅行裡，全程用它管理所有花費。

### 功能

- 自動辨識日文收據，AI 擷取店名、金額、品項、稅別，翻譯成繁體中文
- 即時 Dashboard：今日花費、旅程累計、現金預算進度
- 統計分析：每日趨勢、類別佔比、支付方式分布、TOP 10 消費
- 根據旅程日程，自動判斷消費地區
- 多人記帳，頭像區分
- Notion 作為資料庫，即時同步

## Architecture Overview

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│   Mobile /  │    │  Next.js 16  │    │   Notion    │
│   Browser   │───▶│  App Router  │───▶│  Database   │
│   (PWA)     │    │  API Routes  │    │  (Storage)  │
└─────────────┘    └──────┬───────┘    └─────────────┘
                          │
                   ┌──────▼───────┐
                   │  Gemini 2.0  │
                   │  Flash API   │
                   │  (Vision AI) │
                   └──────────────┘
```

## Core Stack

| Layer | Tech | Why |
|:------|:-----|:----|
| Framework | **Next.js 16** (App Router) | Server-side API routes + client SPA，一個框架搞定前後端 |
| Language | **TypeScript** (strict mode) | 收據資料結構複雜（稅制、多幣別），型別檢查防止計算錯誤 |
| Styling | **Tailwind CSS v4 + 自訂 Design Tokens** | Tailwind utilities 搭配 CSS variables 管理色彩與元件樣式 |
| AI / OCR | **Google Gemini 2.0 Flash** | 圖片辨識 + 日文翻譯 + 結構化輸出，一個 API call 搞定 |
| Database | **Notion API** | 免費、有 UI 可以手動編輯、方便匯出，適合旅行記帳場景 |
| Deployment | **Zeabur** | 自動偵測 Next.js，zero-config 部署 |

## AI — Gemini Vision API

**這是整個專案的核心。** 一張收據照片丟進去，Gemini 回傳完整結構化 JSON：

```
收據照片 → Gemini 2.0 Flash → {
  storeName: "全家便利商店",    // 自動翻譯繁體中文
  storeNameJa: "ファミリーマート", // 保留日文原文
  items: "飯糰, 綠茶",         // 商品翻譯
  itemsJa: "おにぎり, 緑茶",    // 商品原文
  amountJPY: 432,              // 日幣金額
  amountTWD: 90.72,            // 自動換算台幣
  taxType: "内税",              // 辨識日本三種稅制
  category: "餐飲",             // 自動分類
  paymentMethod: "現金",        // 辨識付款方式
  date: "2026-03-01",          // 辨識日期
  region: "名古屋",             // 根據日期自動對應地區
  ...
}
```

### 為什麼選 Gemini？

1. **免費額度超夠用** — 旅行 25 天掃了上百張收據，總花費 $0
2. **多模態能力強** — 圖片辨識 + 日文 OCR + 翻譯 + 分類，一次搞定
3. **結構化輸出** — 直接回傳 JSON，不用再 parse
4. **速度快** — Flash 模型回應快，掃描體驗流暢

### Gemini 定價參考

| Model | Input | Output | Free Tier |
|:------|:------|:-------|:----------|
| Gemini 2.0 Flash | $0.10/1M tokens | $0.40/1M tokens | 1,500 req/day |

我的實際用量（25 天旅行）：~243K tokens，全部落在免費額度內。

### Model Fallback 策略

```typescript
const MODELS = [
  "gemini-2.0-flash-001",  // 穩定版優先
  "gemini-2.0-flash",      // 最新版備援
  "gemini-1.5-flash",      // 最終 fallback
];
```

### Prompt Engineering 重點

Gemini prompt 是這個專案最花時間的部分（~143 行），關鍵挑戰：

- **日本三種稅制**：外税（價格不含稅）、内税（價格含稅）、免税（退稅），計算邏輯不同
- **折扣格式**：割引、値引、各種日文折扣寫法
- **多稅率**：同一張收據可能有 8%（食品）和 10%（非食品）兩種稅率
- **金額驗證**：要求 Gemini 自行驗算，避免 hallucination

## 專案結構

```
src/
├── app/
│   ├── page.tsx                 # Dashboard
│   ├── layout.tsx               # Root layout（metadata, fonts, PWA）
│   ├── globals.css              # Tailwind + 自訂 Design Tokens
│   ├── scan/
│   │   ├── page.tsx             # 拍照 / 上傳
│   │   └── confirm/page.tsx     # 確認 AI 結果 + 手動修正
│   ├── add/page.tsx             # 手動輸入
│   ├── history/page.tsx         # 歷史記錄
│   ├── stats/page.tsx           # 統計分析
│   ├── settings/page.tsx        # 設定
│   └── api/
│       ├── analyze/route.ts     # Gemini Vision API
│       ├── debug/route.ts       # API 健康檢查
│       └── notion/
│           ├── route.ts         # GET/POST 收據記錄
│           ├── items/route.ts   # 逐品項寫入（含稅額分攤）
│           ├── update/route.ts  # 更新記錄
│           └── rename-user/route.ts  # 批次改名
├── lib/
│   ├── types.ts                 # TypeScript 型別定義
│   ├── gemini.ts                # Gemini API wrapper + prompt
│   ├── notion.ts                # Notion API client
│   ├── settings.ts              # 設定管理（localStorage）
│   ├── cache.ts                 # In-memory cache（TTL: 3 分鐘）
│   ├── demo-mode.ts             # Demo 模式切換
│   └── mock-data.ts             # 50+ 筆假資料
└── components/                  # （目前為空，UI 直接寫在 page）
```

## Frontend

### 頁面結構

```
/              → Dashboard（總覽 + 快捷入口）
/scan          → 拍照 / 上傳收據
/scan/confirm  → 確認 AI 辨識結果 + 手動修正
/add           → 手動輸入（沒有收據時）
/history       → 歷史記錄列表（編輯 / 刪除）
/stats         → 統計分析（預算追蹤 / 分類圖表）
/settings      → 設定（預算 / 匯率 / 行程）
```

### State Management

不用 Redux / Zustand，純 React hooks：

- `useState` — 頁面狀態
- `localStorage` — 使用者設定（預算、匯率、行程日期）
- **Notion** — 收據資料的 source of truth

### 預算與行程設定

Settings 頁面讓使用者自訂旅行參數，全部存在 `localStorage`，不需要後端：

```typescript
interface AppSettings {
  budget: number;         // 總預算 (JPY)
  budgetNote: string;     // 預算備註（如：現金 + Suica 明細）
  exchangeRate: number;   // 匯率 (JPY → TWD)
  tripDays: number;       // 旅行天數
  tripSchedule: string;   // 行程表（多行文字）
  user1: string;          // 旅伴 1
  user2: string;          // 旅伴 2
}
```

**預算系統**：
- 設定總預算（日幣），系統自動算出「每日預算」
- 支援備註欄記錄預算組成（例：`287,000 現鈔 + 5,000 Suica + 20,000 Suica = ¥312,000`）
- 匯率即時換算顯示台幣金額（`¥1,000 ≈ NT$206`）
- Stats 頁面追蹤預算消化進度

**行程地區自動判定**：
- 用純文字設定行程，格式：`名古屋 2/23-2/28`（一行一個地區）
- 系統解析日期範圍，掃描收據時根據日期自動歸到對應地區
- 不需要手動選地區，減少操作步驟

**多用戶支援**：
- 設定旅伴名稱，掃描時可選擇「誰付的」
- 改名時自動同步更新所有 Notion 記錄（呼叫 `/api/notion/rename-user`）
- Stats 頁面可按用戶篩選，分開看各自的花費

### PWA

可以加到手機桌面，像原生 App 一樣使用。旅行途中直接開 PWA 掃描收據。

<!-- TODO: 加入 App 圖示截圖 -->

## Backend — API Routes

全部用 Next.js App Router 的 Route Handlers，不需要另外架 server：

| Endpoint | Method | 功能 |
|:---------|:-------|:-----|
| `/api/analyze` | POST | 收據圖片 → Gemini 辨識 |
| `/api/notion` | GET | 取得所有收據記錄（3 分鐘 cache） |
| `/api/notion` | POST | 新增收據到 Notion |
| `/api/notion/items` | POST | 逐品項寫入（含稅額分攤計算） |
| `/api/notion/update` | POST | 更新既有記錄 |
| `/api/notion/rename-user` | POST | 批次改名 |

### Caching

Notion API 每次查詢要 1-2 秒，加了 in-memory cache（TTL: 3 分鐘），寫入時自動 invalidate。

## Database — Notion

### 為什麼用 Notion 當 Database？

1. **免費** — 個人版不限頁面數
2. **有 GUI** — 可以直接在 Notion 裡查看、篩選、排序、手動修正
3. **可匯出** — 隨時匯出 CSV
4. **API 成熟** — `@notionhq/client` SDK 穩定好用

### 串接注意事項

- **分頁處理** — Notion API 每次最多回傳 100 筆，超過要用 `start_cursor` 做分頁迴圈
- **Property 型別對應** — 每種欄位的讀寫格式不同（title、rich_text、number、date、select），寫錯結構就 400 error
- **Schema 相容** — 如果中途改過欄位型別（例如 title → rich_text），讀取時要同時處理兩種結構
- **TWD 換算是 Notion Formula** — 台幣金額不是程式算的，是 Notion 內建 formula：`round(prop("金額 (JPY)") * 0.21)`

### Schema

| 欄位 | 類型 | 說明 |
|:-----|:-----|:-----|
| 項目 | Title | 商品名稱（繁中） |
| 商店名稱 | Rich Text | 店名（繁中） |
| 商店日文 | Rich Text | 店名（日文原文） |
| 商品日文 | Rich Text | 商品（日文原文） |
| 日期 | Date | 消費日期 |
| 金額 (JPY) | Number | 日幣金額 |
| 金額 (TWD) | Formula | 台幣金額（自動換算） |
| 類別 | Select | 餐飲 / 交通 / 購物 / 門票 / 住宿 / 藥品 / 其他 |
| 支付方式 | Select | 現金 / 信用卡 / Suica / PayPay / 其他 |
| 地區 | Select | 名古屋 / 靜岡 / 松本 / 高山 / 金澤 |
| 用戶 | Rich Text | 記帳人 |
| 備註 | Rich Text | 稅制、折扣資訊 |

## 成本

| 項目 | 費用 |
|:-----|:-----|
| Gemini API | **$0**（免費額度） |
| Notion | **$0**（個人版免費） |
| Zeabur | **$5/月**（Developer Plan） |
| 網域 | 不需要（用 `*.zeabur.app`，也可以自綁定網域） |
| **Total** | **$5/月** |

## 如果你也想做

### 取得 API Keys

1. **Gemini** → [Google AI Studio](https://aistudio.google.com/) 建立 API key
2. **Notion** → [Notion Integrations](https://www.notion.so/my-integrations) 建立 integration → 分享 database 給 integration

### 開發

```bash
npm install
npm run dev    # http://localhost:3000
```

### 部署

推上 GitHub → 連接 Zeabur / Vercel → 設定環境變數 → 完成。

## 未來可改善

1. **收據照片沒有保存** — 掃描後圖片就丟了，如果 Gemini 辨識錯誤想回頭對照原圖沒辦法。可以存到 Notion 附件或 Cloudflare R2
2. **圖片沒有壓縮** — 手機拍的收據照片可能 3-5MB，直接 base64 傳給 Gemini。加 client-side resize（壓到 1024px 寬）可以加快上傳，辨識效果不會差
3. **沒有 Auth** — 任何人知道 URL 都能用。個人使用沒問題，但分享給別人用的話需要加 password middleware 或 Zeabur Basic Auth

## 關鍵學習

1. **Prompt Engineering 是最花時間的** — 日本稅制的邊界情況很多，prompt 迭代了十幾版
2. **Gemini Flash 免費額度很夠** — 旅行場景的用量根本用不完
3. **Notion 當 DB 意外好用** — 有 GUI 可以手動修正 AI 辨識錯誤，比傳統 DB 方便
4. **PWA 是旅行 App 的最佳選擇** — 不用上架 App Store，掃 QR code 就能用
5. **不需要 UI Library** — 6 個頁面用 Tailwind 手刻更快，也更輕量
