# Veltix 專案進度 Log

此檔案記錄每次對話的進度，供下一次對話或回來分析時快速接上脈絡。

格式：最新的在最上面，舊的往下堆。

---

## 2026-05-17

### 完成
- Dashboard 範圍定案：只放 EDA 三節，不放 cleaning summary、不放敘述文字、不接 Gemini AI
- 部署形式：純靜態 `index.html` 放 repo 根目錄，Plotly.js 走 CDN，GitHub Pages 從 `main / root` 部署
- 從 `data/cleaned/veltix_cleaned.csv` 一次性計算所有 dashboard 資料（KPI / 3 charts / 3 tables），inline 成 `const DATA = {...}` 嵌進 HTML
- 四張 KPI 卡：Coverage（50 SKUs / 156 weeks）、低周轉 15/50、Worst SKU MA-009、Holding cost gap 6.2× higher
- 三張 Plotly 互動圖：
  1. Turnover ranking bar（50 SKUs 降冪，低周轉紅色其他藍色，4.5× 紅虛線標 industry floor）
  2. CV scatter（15 低周轉 SKU 三群著色，0.3 / 0.6 灰虛線標 bucket 邊界）
  3. SL vs HC 三條折線（單張圖三色，點上直接標 $ 值）
- 三張表用 tab 切換（不並列，避免太窄）：Turnover Ranking 50 / CV Segmentation 15 / Safety Stock Simulation 15
- 修正 SS 表欄位 / 數字偏移（CSS `.num` 原本只套 td 沒套 th，補上 th.num 後對齊）
- KPI 文案 polish：Worst SKU 主角改成 SKU id（不是 turnover 數字）、Cost gap 加上 `higher` 跟 `$33/wk vs $5/wk` 錨點、`@` 改成括號展開 SL → service level
- 數字對帳：50 SKUs turnover 排序、CV 分群、HC 三組均值、6.2× 比值都跟 EDA notebook 對得起來

### 下一步
- 把 `index.html` push 上去
- GitHub repo 設定 → Pages → source 選 `main / root`，等部署完拿到公開 URL
- 之後想加 README badge 連到 dashboard URL

### 備註 / 卡關
- 早期討論時用戶推翻 PRD 的「Executive Summary + Gemini AI 報告」設計，改成純可互動展示
- pipeline.py 同步決定不寫了（先前 PRD §9 規劃由 AI 從 notebook 生成的 pipeline.py 暫不做）
- output.png 是稍早隨便丟的截圖，沒 commit
- docs/Veltix PRD.md 在工作目錄被刪了（pre-existing change），先不動，留給用戶決定要不要一起 commit

---

## 2026-05-10

### 完成
- 檔案改名 `data_profiling.ipynb` → `data_cleaning.ipynb`
- 結構重新組織：原本「按檢查類型橫向組織（2.4 單欄位 / 2.5 跨欄位）」改成「按處理流程縱向組織」：
  1. Data Overview
  2. Key & Category 欄位清理
  3. Duplicates / Missing / Invalid 檢查
- Section 1 Overview：shape / info / describe / nunique，鎖定 cardinality 問題（sku 241→應 50、category 35→應 5、week 277→應 156）
- Section 2 Key & Category 全部清完：
  - 三欄一起去前後空白
  - `product_category`：title case → abbreviation mapping（Mob. Acc. → Mobile Accessories 等）→ 二輪 mapping（Smart Home Devices → Smart Home 等）→ 5 個正確值
  - `sku_id`：upper / `_`→`-` / `O`→`0` / `RNA-`→`MA-`（用 unit_cost 交叉驗證確認同產品）/ `L`→`1`（同樣用 unit_cost 驗證 L 是視覺混淆）→ 50 個正確值
  - `week_id`：`/`→`-` / 反向格式 W45-2025→2025-W45 / 單位數週補 leading zero（W3→W03）→ 156 個正確值
- Section 3 Duplicates：用 `df.duplicated().sum() == df.duplicated(subset=keys).sum()` 確認 234 筆是完整重複，drop_duplicates → 7800 筆（對上 dictionary 50×156）
- Section 3 Missing values 盤點：sales_qty 266、inventory_begin 275，兩者無重疊 → 決定先驗庫存等式再回頭用等式回推
- Section 3 Invalid logic：庫存等式驗證 219 mismatch → 拆 sign error (68) / genuine (151)
- Sign error 修完：68 筆 + 7 筆漏網負值（5 筆 inventory_begin NaN 沒進等式檢查、2 筆已歸 genuine）一次翻正，全 df 無負值
- Genuine error 處理：先用 describe + hist 看誤差分布（median −39、IQR ±200，但 max 3177 為極端值），再按 category 與 week 分布確認沒有集中性 → 直接 drop 151 筆
- Missing value imputation：用等式移項回推（缺 sales_qty 用 `inv_begin + receipts - inv_end`、缺 inv_begin 用 `inv_end + sales_qty - receipts`）`fillna(Series)` 補完
- Lead time logic check（補上原本只在 describe 觀察、未實際處理的問題）：用 `sorted(unique())` 攤開 53 個值，發現 33 筆落在 (1, 13) 違反「有進貨才有 lead time」業務規則、2 筆 > 56 → 全 drop 35 筆
- Final validation：發現 imputation 後仍漏 1 筆 negative sales_qty（imputation 暴露出 NaN 列其他三欄的隱性 data integrity 問題）→ drop 補上
- 最終 7613 筆，存到 `data/cleaned/veltix_cleaned.csv`
- Commit `c3f3afb` 已 push 到 origin/main

### 下一步
- 進入 `notebooks/EDA.ipynb`（已建空檔）開始探索性分析
- 之後接 `pipeline.py` 由 AI 從 cleaning + EDA notebook 生成

### 備註 / 卡關
- 原本 4/12 規劃的「按檢查類型橫向組織」結構已放棄，改成按處理流程組織
- 原規劃的 unit_cost / holding_cost_rate 範圍檢查在 describe 觀察階段已確認 OK；其他 4 項跨列一致性（庫存連續性、unit_cost 固定性、rate 固定性、lead_time 有效性）這版 cleaning notebook 沒做，留到 EDA 或後續再補
- log 距上次更新近一個月，跟 notebook 實際狀態落差很大，這次大幅改寫對齊現況

---

## 2026-04-12

### 完成
- 建立 `log.md` 進度追蹤機制
- 在 `CLAUDE.md` 新增規則：每次對話結束時自動更新本檔
- 對照 `veltix_data_dictionary.md` review `data_profiling.ipynb`，找出缺口：
  - 數值異常只做 sales_qty，漏 inventory_begin/end、receipts、lead_time_days、unit_cost、holding_cost_rate
  - 邏輯一致性只做 1/5（庫存等式），漏 lead_time 有效性、庫存連續性、unit_cost 固定性、rate 固定性
- 釐清 profiling vs EDA 差別：檔名 `data_profiling.ipynb` 正確，EDA 應另開 notebook 在清理後做
- 重構 Data Quality Check 結構：把「數值異常（2.4 單欄位）」和「邏輯一致性（2.5 跨欄位/跨列）」拆成兩個獨立區塊

### 進行中
- 補 2.4 數值異常：6 個欄位已列出檢查規則表
  - inventory_begin / end / receipts：查負值
  - lead_time_days：不在 {0} ∪ [14,56]
  - unit_cost：不在 [3, 140]
  - holding_cost_rate：不在 [0.25, 0.30]
- 正在教效率寫法：多欄位負值一次算完（不要寫 3 行重複 code）

### 下一步
- 完成 2.4 多欄位負值 vectorized 寫法
- 續寫 lead_time_days / unit_cost / holding_cost_rate 範圍檢查
- 進入 2.5 邏輯一致性 5 項（庫存等式已完成、lead_time 有效性、庫存連續性、unit_cost 固定性、rate 固定性）

### 備註 / 卡關
- 教學中發現：用戶把「> 0」和「>= 0」混用，inventory 可以 = 0（賣光）不是異常
- 用戶採「按檢查類型組織」結構（橫向：一個檢查掃多欄位），所以每類檢查要系統性覆蓋該掃的欄位子集
