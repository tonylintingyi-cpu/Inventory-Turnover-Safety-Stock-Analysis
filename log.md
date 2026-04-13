# Veltix 專案進度 Log

此檔案記錄每次對話的進度，供下一次對話或回來分析時快速接上脈絡。

格式：最新的在最上面，舊的往下堆。

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
