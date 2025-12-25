# nfc-timing-2025_V2

https://github.com/winner1628/nfc-timing-2025_V2
<img width="230" height="230" alt="qrcode" src="https://github.com/user-attachments/assets/5eeb6420-943c-4c51-885c-ab35460f11ae" />

https://winner1628.github.io/nfc-timing-2025_V2/timing-adminv2.html
<img width="230" height="230" alt="qrcode (1)" src="https://github.com/user-attachments/assets/d21c8f3d-e755-4e83-8ef7-3c19ae955325" />

https://winner1628.github.io/nfc-timing-2025_V2/timing-phonev2.html
<img width="230" height="230" alt="qrcode (2)" src="https://github.com/user-attachments/assets/6b9880f8-1e8d-4665-9305-433a6c74e96f" />


# NFC 跑手計時系統 - 專案總結（修正版）
## 總覽
此專案為基於 Supabase（PostgreSQL）後端資料庫與原生 JavaScript/HTML/CSS 前端開發的**網頁式 NFC 跑手計時管理系統**，主要功能包含追蹤跑手在賽事檢查點的打卡紀錄、計算賽事用時、生成排名，以及提供後台管理能力（匯入/匯出數據、參賽者管理）。

---

## 資料庫架構（Supabase/PostgreSQL）
### 1. 核心資料庫：Supabase（PostgreSQL）
所有數據儲存於 Supabase（託管式 PostgreSQL），並透過即時訂閱功能實現打卡事件的即時更新。

### 2. 完整資料庫表格（DDL 建立語句）
確認實際使用的表格共 **4 個**（修正先前疏漏），以下為完整定義：

#### 表格 1：`participants`（賽事參賽者）
儲存跑手基本資訊（參賽號碼、姓名）
```sql
CREATE TABLE public.participants (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  bib_number TEXT NOT NULL UNIQUE, -- 參賽號碼（唯一識別）
  name TEXT NOT NULL, -- 參賽者姓名
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(), -- 建立時間
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() -- 更新時間
);

-- 加速參賽號碼查詢的索引
CREATE INDEX idx_participants_bib_number ON public.participants(bib_number);
```

#### 表格 2：`checkins`（打卡紀錄）
儲存所有 NFC 打卡事件（跑手在各檢查點的打卡紀錄）
```sql
CREATE TABLE public.checkins (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  bib_number TEXT NOT NULL, -- 對應 participants.bib_number
  checkpoint TEXT NOT NULL, -- 檢查點代碼（START/FINISH/CP1/CP2...）
  time TIMESTAMP WITH TIME ZONE DEFAULT NOW(), -- 打卡時間
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(), -- 紀錄建立時間
  
  -- 外鍵約束（確保參照完整性）
  CONSTRAINT fk_checkins_bib_number 
    FOREIGN KEY (bib_number) 
    REFERENCES public.participants(bib_number)
    ON DELETE CASCADE -- 刪除參賽者時自動刪除對應打卡紀錄
);

-- 優化查詢效能的索引
CREATE INDEX idx_checkins_bib_number ON public.checkins(bib_number);
CREATE INDEX idx_checkins_checkpoint ON public.checkins(checkpoint);
CREATE INDEX idx_checkins_time ON public.checkins(time);
```

#### 表格 3：`checkpoint_configs`（檢查點配置）
儲存賽事檢查點的元數據（可透過後台介面配置）
```sql
CREATE TABLE public.checkpoint_configs (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  event_id TEXT DEFAULT 'default' NOT NULL, -- 賽事ID（支援多賽事管理）
  code TEXT NOT NULL, -- 檢查點代碼（START/FINISH/CP1...）
  name TEXT NOT NULL, -- 檢查點名稱（起點/終點/檢查點1...）
  order_index INTEGER NOT NULL, -- 檢查點順序
  is_required BOOLEAN DEFAULT TRUE, -- 是否為必經檢查點
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  
  -- 賽事+檢查點代碼唯一約束
  CONSTRAINT unique_event_checkpoint_code 
    UNIQUE (event_id, code)
);

-- 索引優化
CREATE INDEX idx_checkpoint_configs_event_id ON public.checkpoint_configs(event_id);
CREATE INDEX idx_checkpoint_configs_order_index ON public.checkpoint_configs(order_index);
```

#### 表格 4：`race_stats`（賽事統計）
**先前疏漏的表格**，用於儲存賽事整體統計數據（減少前端即時計算壓力）
```sql
CREATE TABLE public.race_stats (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  event_id TEXT DEFAULT 'default' NOT NULL, -- 對應賽事ID
  total_runners INTEGER DEFAULT 0, -- 總參賽人數
  started_runners INTEGER DEFAULT 0, -- 已出發人數
  finished_runners INTEGER DEFAULT 0, -- 已完賽人數
  avg_finish_time INTEGER DEFAULT 0, -- 平均完賽時間（秒）
  last_updated TIMESTAMP WITH TIME ZONE DEFAULT NOW(), -- 最後更新時間
  
  -- 單一賽事僅保留一筆統計紀錄
  CONSTRAINT unique_race_stats_event_id 
    UNIQUE (event_id)
);

-- 索引優化
CREATE INDEX idx_race_stats_event_id ON public.race_stats(event_id);
```

### 3. Supabase 即時功能設定
啟用 `checkins`、`participants` 與 `race_stats` 表格的即時訂閱，實現前端即時更新：
```sql
-- 啟用公開結構的即時發布
ALTER PUBLICATION supabase_realtime 
ADD TABLE public.checkins, public.participants, public.race_stats, public.checkpoint_configs;

-- 授予匿名/已驗證使用者權限
GRANT SELECT, INSERT, UPDATE, DELETE ON public.participants TO anon, authenticated;
GRANT SELECT, INSERT, UPDATE, DELETE ON public.checkins TO anon, authenticated;
GRANT SELECT, INSERT, UPDATE, DELETE ON public.checkpoint_configs TO anon, authenticated;
GRANT SELECT, INSERT, UPDATE, DELETE ON public.race_stats TO anon, authenticated;
```

---

## 資料庫核心特性
### 1. 數據關聯性
- **一對多關係**：一個參賽者（bib_number）對應多筆打卡紀錄
- **級聯刪除**：刪除參賽者時自動刪除其所有打卡紀錄
- **參照完整性**：外鍵約束防止無效的參賽號碼出現在打卡紀錄中
- **唯一約束**：確保參賽號碼、賽事+檢查點代碼、賽事統計紀錄不重複

### 2. 索引策略
| 索引名稱 | 對應表格 | 用途 |
|----------|----------|------|
| `idx_participants_bib_number` | `participants` | 快速透過參賽號碼查詢跑手資訊 |
| `idx_checkins_bib_number` | `checkins` | 快速查詢特定跑手的所有打卡紀錄 |
| `idx_checkins_checkpoint` | `checkins` | 快速篩選特定檢查點的打卡紀錄（如起點/終點） |
| `idx_checkins_time` | `checkins` | 快速按時間排序/篩選打卡紀錄 |
| `idx_checkpoint_configs_event_id` | `checkpoint_configs` | 快速查詢特定賽事的檢查點配置 |
| `idx_race_stats_event_id` | `race_stats` | 快速查詢特定賽事的統計數據 |

### 3. 預設值與時間戳
- 自動生成 `created_at`/`updated_at` 時間戳，便於數據審計
- `event_id` 預設為 `default`，支援單一賽事場景
- `is_required` 預設為 `TRUE`，確保起點/終點為必經檢查點
- 統計數據預設值為 0，避免空值處理問題

---

## 系統核心功能（資料庫驅動）
### 1. 參賽者管理
- **匯入**：透過 CSV 批量匯入參賽者（格式：bib_number, name）
- **CRUD 操作**：建立/查詢/更新/刪除參賽者紀錄
- **唯一性驗證**：參賽號碼唯一約束防止重複

### 2. 打卡追蹤
- **即時更新**：透過 Supabase Realtime 訂閱，新增打卡紀錄時前端自動更新
- **用時計算**：基於起點/終點打卡時間計算完賽用時
- **多維度篩選**：可按檢查點、參賽號碼、時間範圍查詢打卡紀錄

### 3. 排名與統計
- **多種排名模式**：
  - 槍時（按終點到達時間）
  - 晶片時（按實際用時）
  - 分段成績（按各檢查點用時）
- **儀表板指標**：
  - 總參賽人數、已出發人數、已完賽人數
  - 平均完賽時間
  - 賽事進度視覺化（Chart.js 圓環圖）
- **統計快取**：`race_stats` 表格儲存計算結果，減少前端即時計算壓力

### 4. 數據匯入/匯出
- **CSV 匯出**：匯出所有賽事數據（總覽、排名、打卡紀錄）
- **CSV 匯入**：批量匯入參賽者
- **數據清空**：管理員專用批量刪除功能（雙重確認防止誤操作）

---

## 技術要求
### 1. Supabase 設定
- 建立 Supabase 專案（免費方案可用）
- 執行上述 DDL 語句建立表格與索引
- 啟用 `checkins`、`participants`、`race_stats`、`checkpoint_configs` 的即時功能
- 取得 Supabase URL 與 anon key 用於前端連接

### 2. 權限建議
- **開發環境**：匿名角色給予完整 CRUD 權限（如範例所示）
- **生產環境**：啟用列級安全性（RLS），限制匿名角色僅可讀取，管理員需驗證後操作

### 3. 最佳實踐
- **備份**：啟用 Supabase 自動備份功能
- **效能優化**：大數據量（1000+ 跑手）時確保索引全部建立
- **擴展性**：透過 `event_id` 欄位實現多賽事數據隔離
- **清理策略**：新增定時腳本清理舊賽事數據（按需設定）

此架構適用於中小型賽事（100-1000 名跑手），針對大型賽事可額外增加索引、分表或快取機制進一步優化。

NFC 跑手計時系統 (NFC Race Timing System)
一個基於網頁的全方位賽事計時與管理系統，採用 NFC 技術追蹤參賽者打卡記錄，後端依託 Supabase (PostgreSQL) 進行數據儲存與即時更新，透過 Chart.js 實現數據視覺化，並以原生 JavaScript 開發前端介面，專為賽事籌辦方設計。
📋 專案概述
本系統為賽事主辦單位提供完整解決方案，可即時追蹤參賽者各檢查點打卡記錄、管理排名、視覺化賽事進度。支援多檢查點設定、CSV 匯入匯出功能，並具備管理員級別的數據管理介面，擁有響應式設計，適用於桌面與行動裝置。
核心功能
功能	說明
🚀 即時追蹤	透過 Supabase Realtime 即時更新參賽者打卡記錄
🏆 多模式排名	支援依槍時間、晶片時間或各檢查點分段成績排名
📊 賽事儀表板	透過圖表視覺化賽事進度（未開始 / 進行中 / 已完賽人數）
👥 參賽者管理	新增 / 編輯 / 刪除參賽者、CSV 批次匯入匯出
🚩 檢查點配置	自訂賽事檢查點（必填 / 非必填、排序設定）
📱 響應式設計	行動裝置友善介面，最佳化各尺寸螢幕顯示
🔒 數據安全	重要操作需確認提示，刪除邏輯符合安全規範
📝 操作日誌	即時記錄所有系統操作事件，便於追蹤與除錯

🛠️ 技術架構
前端
原生 JavaScript (ES6+)
HTML5/CSS3（響應式設計、客製化樣式）
Chart.js v4（數據視覺化，賽事進度環狀圖）
CSS Flexbox/Grid 響應式網格佈局
後端 / 資料庫
Supabase (PostgreSQL) 數據儲存
Supabase Realtime 即時更新功能
Supabase REST API 執行 CRUD 操作
核心技術實現
安全刪除機制：實作符合 Supabase 規範的 DELETE 操作（必須包含 WHERE 子句）
即時訂閱：監聽資料庫變更（INSERT/DELETE）並即時更新介面
時間計算：精確計算各檢查點間耗時（格式化為 HH:MM:SS）
CSV 處理：匯入匯出支援 UTF-8 BOM，解決中文亂碼問題
自動刷新：後台每 5 秒自動刷新數據，同時保留手動刷新選項

📁 資料庫結構
核心資料表
participants（參賽者）
欄位	類型	說明
bib_number	TEXT (主鍵)	參賽者唯一編號（號碼布）
name	TEXT	參賽者姓名
checkins（打卡記錄）
欄位	類型	說明
id	UUID (主鍵)	自動生成的唯一識別碼
bib_number	TEXT	關聯至參賽者的外部鍵
checkpoint	TEXT	檢查點代碼（START/FINISH/CP1 等）
time	TIMESTAMP	精確打卡時間戳記
checkpoint_configs（檢查點配置）
欄位	類型	說明
code	TEXT	檢查點唯一代碼
name	TEXT	顯示名稱（如：起點、終點）
order_index	INTEGER	賽事中的順序編號
is_required	BOOLEAN	是否為必經檢查點
event_id	TEXT	賽事識別碼（預設："default"）

🚀 快速開始
先備條件
Supabase 帳戶與專案
Supabase URL 與 anon/public 金鑰
基礎網頁伺服器（或直接在瀏覽器開啟 HTML 檔）
設定步驟
複製儲存庫
bash
运行
git clone [你的儲存庫URL]
cd nfc-race-timing
設定 Supabase
建立上述資料庫結構中的資料表
從專案設定取得 Supabase URL 與 anon 金鑰
執行應用程式
在現代瀏覽器開啟 HTML 檔
在管理面板輸入 Supabase URL 與金鑰
點擊「連線後台」按鈕
設定賽事參數
點擊「👥 管理參賽者」設定檢查點與參賽者資料
透過 CSV 匯入或手動新增參賽者
使用說明
儀表板視圖：監控整體賽事進度與核心統計數據
排名視圖：依不同指標檢視詳細排名
打卡記錄視圖：查看所有檢查點記錄，支援篩選功能
數據管理：匯入 / 匯出數據或清空所有記錄（需確認）

🎨 介面特色
獎牌高亮：前三名參賽者以金 / 銀 / 銅色背景標示
即時日誌：記錄所有系統事件（新打卡、數據變更等）
統計卡片：核心指標展示（總人數 / 已出發 / 已完賽人數、平均完賽時間）
互動式圖表：視覺化賽事進度，支援滑鼠懸停顯示詳細資訊
彈窗介面：整潔的參賽者管理表單，含表單驗證

⚠️ 重要注意事項
數據安全：所有刪除操作需二次確認，防止誤刪數據
Supabase 設定：請依使用情境適當配置列級安全性（RLS）
備份建議：定期透過 CSV 匯出功能備份數據
時區設定：系統使用瀏覽器當地時區顯示時間
效能最佳化：僅載入最近 100 筆打卡記錄以確保效能

🌐 在地化支援
介面語言：繁體中文（香港）
時間格式：24 小時制（HH:MM:SS）
日期格式：DD/MM/YYYY（香港標準）
支援 UTF-8 編碼，確保中文字元正常顯示

🛡️ 安全考量
所有敏感操作（刪除、清空全部）均需確認提示
Supabase 憑證儲存在 localStorage（公開部署時不建議使用）
原始程式碼中無硬編碼憑證
刪除操作符合 Supabase 安全規則

📱 行動裝置支援
響應式網格佈局，適配行動裝置螢幕
適合觸控操作的按鈕與表單元素
最佳化的表格滾動體驗
針對行動裝置最佳化的圖表顯示

🤝 貢獻方式
Fork 本儲存庫
建立功能分支 (git checkout -b feature/新功能)
提交變更 (git commit -m '新增某項功能')
推送至分支 (git push origin feature/新功能)
開啟 Pull Request

📄 授權條款
本專案採用開源授權，基於 MIT 授權條款 釋出。

📸 截圖展示（選擇性）
(在此處新增系統截圖，展示介面特色)
儀表板視圖 - 整體賽事統計與進度圖表
排名視圖 - 詳細參賽者排名，含獎牌高亮效果
打卡記錄視圖 - 可篩選的檢查點記錄列表
參賽者管理 - CSV 匯入與手動新增介面

✨ 未來優化方向
新增賽事分類（年齡組、賽事類型）
實作使用者身分驗證，支援多管理員存取
新增賽事成績列印功能
整合 NFC 硬體 API
新增賽事路線地圖視覺化
實作進階分析與報表功能
