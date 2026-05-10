# Veltix 專案進度 Log

此檔案記錄每次對話的進度，供下一次對話或回來分析時快速接上脈絡。

格式：最新的在最上面，舊的往下堆。

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

### 進行中
- Genuine error 處理（151 筆即使翻正號也無法平衡的庫存等式異常）

### 下一步
- 處理 genuine error
- 回頭用庫存等式回推 missing values（sales_qty 266、inventory_begin 275）

### 備註 / 卡關
- 原本 4/12 規劃的「按檢查類型橫向組織」結構已放棄，改成按處理流程組織
- 原規劃的 lead_time_days / unit_cost / holding_cost_rate 範圍檢查、以及 2.5 剩 4 項邏輯一致性（lead_time 有效性、庫存連續性、unit_cost 固定性、rate 固定性）都還沒做，待確認是否要在 cleaning notebook 補回，還是移到後續 EDA notebook
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
