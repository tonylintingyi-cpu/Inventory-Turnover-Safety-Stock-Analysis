
## 1. Project Overview

**Company:** Veltix（消費性電子產品分銷商，從亞洲進口）

**Data:** 合成資料，50 SKU × 156 週 = 7,800 筆（注入髒數據後約 7,995 筆）

**Data Quality:** 含約 26% 髒數據

**Purpose:** 供應鏈數據分析 Portfolio 專案，展示從資料清洗到庫存優化的完整分析能力

---

## 2. Business Problem

Veltix 的庫存中有一批 SKU 長期賣不動，佔用資金卻沒有貢獻周轉。這些 SKU 不能直接清掉，因為有些只是需求不穩定，不是真的沒人買。加上上游前置時間長（亞洲進口），備貨決策難以拿捏——備少了缺貨，備多了積壓。

**產業參考：** 消費電子分銷商的年庫存周轉率通常落在 4.5–9 次。本專案使用合成資料，實際門檻採雙重條件篩選，產業數字作為設計依據。

---

## 3. Core Analysis Questions

1. 50 個 SKU 的庫存周轉率分布如何？哪些 SKU 資金效率最差？
2. 低周轉 SKU 的需求波動程度如何？依 CV 分成低 / 中 / 高三群。
3. 不同 CV 群組在不同服務水準下，安全庫存與持有成本的關係為何？

---

## 4. Analysis Scope

- **時間範圍：** 2023-W01 ~ 2025-W52（156 週，全部一起算）
- **分析對象：** 50 SKU，5 大產品類別
- **分析單位：** 以 SKU 為單位彙總

---

## 5. Methodology

### Phase 1 — Inventory Turnover Analysis

**目的：** 識別資金效率最差的 SKU。

**公式：**

$$Turnover = \frac{\sum sales_qty}{\overline{inventory}}$$

- 分子：該 SKU 在 156 週內的總銷售量
- 分母：該 SKU 每週平均庫存，計算方式為每週 $(inventory_begin + inventory_end) / 2$，再取 156 週的簡單平均

**低周轉門檻：** 雙重條件 — 年周轉率 < 4.5（低於產業下限）且排名後 30%（約 15 個 SKU）。產業參考值為消費電子分銷商年周轉 4.5–9 次。

**合成資料設計：** 正常 SKU 的銷售量與庫存水位比例設計為年周轉率落在 4.5–9 次；低周轉 SKU（約 15 個）設計為年周轉率 < 2 次，模擬庫存堆積但銷售低迷的情境。

---

### Phase 2 — Demand Volatility Analysis

**目的：** 對低周轉 SKU 進一步分群，區分「需求穩定但就是賣不動」vs「需求不規則導致難以備貨」。

**對象：** Phase 1 篩出的低周轉 SKU（約 15 個）

**公式：**

$$CV = \frac{\sigma_d}{\bar{d}}$$

- $\sigma_d$：該 SKU 156 週銷售量的標準差
- $\bar{d}$：該 SKU 156 週銷售量的均值

**假設：** CV 計算納入所有週次，包含零銷售週。零需求反映真實的需求不確定性，排除會低估波動性。

**分群門檻：**

| 群組 | CV 範圍 | 意義 |
| --- | --- | --- |
| 低波動 | CV < 0.3 | 需求相對穩定，可用固定安全庫存策略 |
| 中波動 | 0.3 ≤ CV ≤ 0.6 | 需求有一定波動，需要彈性備貨 |
| 高波動 | CV > 0.6 | 需求很不規則，備貨風險最高 |

---

### Phase 3 — Safety Stock Simulation

**目的：** 對每個 CV 群組，模擬不同服務水準下的安全庫存與持有成本，找出平衡點。

**安全庫存公式：**

$$SS = Z \times \sqrt{\bar{L} \cdot \sigma_d^2 + \bar{d}^2 \cdot \sigma_L^2}$$

- $Z$：服務水準對應的 Z-score
- $\bar{L}$：該 SKU 平均前置時間（天），僅計算有補貨的週（排除 `lead_time_days = 0`）
- $\sigma_d$：週銷售量標準差
- $\bar{d}$：週銷售量均值
- $\sigma_L$：前置時間標準差，僅取 `lead_time_days > 0` 的週計算

**持有成本公式：**

$$HC = SS \times unit_cost \times \frac{holding_cost_rate}{52}$$

- 年持有成本率除以 52 轉換為週成本

**模擬服務水準：** 85% / 90% / 95%

**輸出：** 持有成本 vs 服務水準曲線，三個 CV 群組各一條線。

---

## 6. Assumptions & Design Decisions

| 項目 | 決策 | 理由 |
| --- | --- | --- |
| 周轉率分母 | 週平均庫存（156 週平均） | 兩點平均無法反映季節性波動 |
| 低周轉門檻 | 年周轉率 < 4.5 且排名後 30% | 4.5 為產業下限，雙重條件確保篩選既有產業依據又數量合理 |
| 合成資料周轉率範圍 | 正常 SKU 4.5–9 次/年，低周轉 SKU < 2 次/年 | 對齊消費電子分銷商產業 benchmark |
| CV 零銷售週 | 納入計算 | 零需求是波動的一部分 |
| CV 分群門檻 | 0.3 / 0.6（保守設定） | 文獻常見分界為 0.2 / 0.7，本專案採較窄切點以確保各群組有足夠樣本數進行 Phase 3 模擬 |
| lead_time_days = 0 | 代表該週無補貨 | 計算 σ_L 時排除 |
| unit_cost | 每個 SKU 各自不同 | 分銷商同類別內產品成本差異大 |
| holding_cost_rate | 依類別不同（0.25–0.30） | 不同類別的跌價風險不同 |

---

## 7. Data Design

詳見 `veltix_data_dictionary.md` 和 `veltix_data_generation_prompt.md`。以下為摘要：

### 7.1 欄位清單

`sku_id`, `product_category`, `week_id`, `sales_qty`, `inventory_begin`, `inventory_end`, `receipts`, `lead_time_days`, `unit_cost`, `holding_cost_rate`

### 7.2 公式 → 欄位對應

| 公式變數 | 來源欄位 | 計算方式 |
| --- | --- | --- |
| Σsales_qty | sales_qty | 156 週加總 |
| avg_inventory | inventory_begin, inventory_end | 每週 (begin+end)/2，取 156 週平均 |
| σ_d | sales_qty | 156 週標準差（含零銷售週） |
| d̄ | sales_qty | 156 週均值（含零銷售週） |
| L̄ | lead_time_days | 排除 0 值後取平均 |
| σ_L | lead_time_days | 排除 0 值後取標準差 |
| unit_cost | unit_cost | 直接取值（每 SKU 固定） |
| holding_cost_rate | holding_cost_rate | 直接取值（依類別固定） |

### 7.3 關鍵生成邏輯

- **周轉率控制**：透過銷售量 μ 與庫存水位的比例，確保正常 SKU 年周轉 4.5–9 次、低周轉 SKU < 2 次
- **低周轉 SKU**：15 個，每類 3 個，銷量降至 25–30%、庫存維持高水位
- **補貨頻率**：正常 SKU 每 4–6 週、低周轉 SKU 每 8–12 週
- **lead_time_days**：receipts > 0 時從常態分布生成；receipts = 0 時填 0（代表無補貨）
- **CV 分群**：不人為指定，由各類別的季節性特徵自然產生
- **髒數據**：乾淨資料生成完成後，逐步注入 8 種類型，總比例約 26%

---

## 8. Deliverables

### 8.1 Data Outputs

| 產出物 | 格式 | 說明 |
| --- | --- | --- |
| Cleaned Dataset | CSV | 清洗後的完整資料集 |
| Issue Log | CSV/JSON | 清洗前的問題記錄（問題類型、影響列數） |
| Summary Stats | CSV/JSON | 清洗後的資料狀態摘要（各欄位缺失率、重複數、值域等） |
| SKU Turnover Summary | CSV | 50 個 SKU 的周轉率排名 |
| CV Segmentation Table | CSV | 低周轉 SKU 的 CV 值與分群結果 |
| Safety Stock Simulation Results | CSV/JSON | 每個 SKU 在 85%/90%/95% 服務水準下的 SS 和 HC |

### 8.2 HTML Report

**格式：** 單一 `report.html` 檔，部署至 GitHub Pages，面試官透過連結直接瀏覽

**優點：** 點擊即可檢視，不需安裝任何軟體；Plotly 提供開箱即用的互動功能（zoom、pan、hover tooltip、圖片匯出）。

**做法：** Python 使用 `plotly` 建立圖表物件，透過 `fig.to_json()` 匯出為 Plotly 原生 JSON（data + layout），搭配 Jinja2 模板引擎注入 HTML。瀏覽器端載入 Plotly.js CDN（~3.5MB，partial bundle ~1MB）並以 `Plotly.newPlot()` 渲染。

**部署：** `report.html` 放在 repo 的 `docs/` 或 `gh-pages` branch，啟用 GitHub Pages 後產生公開 URL，嵌入 README badge 供面試官一鍵存取。

**Python → HTML 數據接口：**

Python 輸出一個 JSON 物件，Jinja2 將其注入 HTML 模板的 `<script>` 區塊，瀏覽器端以 `Plotly.newPlot(div, data, layout)` 渲染：

```json
{
  "report": {
    "executive_summary": "...(Gemini 生成的摘要 HTML)...",
    "technical_report": "...(Gemini 生成的技術報告 HTML)..."
  },
  "charts": {
    "turnover": {
      "data": [{
        "type": "bar",
        "x": ["SKU-001", "SKU-002", "..."],
        "y": [8.2, 7.5, "..."],
        "marker": {"color": ["#4CAF50", "#4CAF50", "...", "#F44336", "..."]},
        "name": "Annual Turnover",
        "hovertemplate": "SKU: %{x}<br>Turnover: %{y:.1f}<extra></extra>"
      }],
      "layout": {
        "title": {"text": "SKU Inventory Turnover Rate"},
        "yaxis": {"title": "Annual Turnover"},
        "shapes": [{"type": "line", "x0": 0, "x1": 1, "xref": "paper", "y0": 4.5, "y1": 4.5, "line": {"dash": "dash", "color": "#F44336"}}]
      }
    },
    "cv_scatter": {
      "data": [
        {"type": "scatter", "mode": "markers", "name": "低波動", "x": [120, "..."], "y": [0.15, "..."], "text": ["SKU-031", "..."], "hovertemplate": "%{text}<br>Demand: %{x}<br>CV: %{y:.2f}<extra></extra>"},
        {"type": "scatter", "mode": "markers", "name": "中波動", "x": [85, "..."], "y": [0.45, "..."], "text": ["SKU-042", "..."]},
        {"type": "scatter", "mode": "markers", "name": "高波動", "x": [40, "..."], "y": [0.78, "..."], "text": ["SKU-017", "..."]}
      ],
      "layout": {
        "title": {"text": "Demand Volatility Segmentation"},
        "xaxis": {"title": "Avg Weekly Demand"},
        "yaxis": {"title": "CV"}
      }
    },
    "tradeoff": {
      "data": [
        {"type": "scatter", "mode": "lines+markers", "name": "低波動", "x": ["85%", "90%", "95%"], "y": [12.5, 18.3, 28.7]},
        {"type": "scatter", "mode": "lines+markers", "name": "中波動", "x": ["85%", "90%", "95%"], "y": [25.1, 36.8, 57.2]},
        {"type": "scatter", "mode": "lines+markers", "name": "高波動", "x": ["85%", "90%", "95%"], "y": [48.3, 70.9, 110.5]}
      ],
      "layout": {
        "title": {"text": "Safety Stock Holding Cost vs Service Level"},
        "xaxis": {"title": "Service Level"},
        "yaxis": {"title": "Avg Weekly Holding Cost ($)"}
      }
    }
  },
  "tables": {
    "turnover": [
      {"sku_id": "SKU-001", "product_category": "Audio", "annual_turnover": 8.2, "is_low_turnover": false},
      "..."
    ],
    "cv_segmentation": [
      {"sku_id": "SKU-031", "product_category": "Audio", "cv_value": 0.15, "cv_group": "低波動", "avg_weekly_demand": 120, "demand_std": 18},
      "..."
    ],
    "safety_stock": [
      {"cv_group": "低波動", "service_level": "85%", "avg_safety_stock": 150, "avg_holding_cost_weekly": 12.5},
      "..."
    ]
  }
}
```

**HTML 報告結構：**

- **Executive Summary 區塊：** Gemini API 生成的摘要與行動建議
- **Dashboard 區塊：** Plotly 互動式圖表（周轉率分布圖、CV 分群散佈圖、安全庫存 tradeoff 曲線），支援 zoom / pan / hover / 圖片匯出
- **Technical Report 區塊：** 完整技術報告，含數據細節與圖表說明
- **Data Tables 區塊：** 上述三張數據表（HTML table，可排序）

---

## 9. Technical Architecture

### 9.1 三層架構（Artifact Layers）

專案產出分為三層，職責嚴格分離：

| 層 | 檔案 / 介面 | 職責 | 不做什麼 |
| --- | --- | --- | --- |
| 探索層 | `notebooks/data_profiling.ipynb`、`notebooks/eda.ipynb` | 展示 profiling 與 EDA 的思考過程，作為清洗規則與分析決策的推導依據 | 不參與 production 執行；不是 pipeline 的一部分 |
| 核心層 | `pipeline.py` | 清洗 + Gemini validation + 三階段分析 + 組裝 HTML 報告 | 不處理觸發邏輯、不做 UI |
| Demo 層 | Claude Skill | 對外 demo 的觸發介面，使用者丟 csv 進對話即呼叫 `pipeline.py` | 不複製分析邏輯；只負責 I/O 介接 |

Skill 呼叫 pipeline，不複製邏輯；notebook 只是思考過程的凍結快照，不會在每次執行時跑。

### 9.2 Pipeline Overview

```
Raw Data (CSV)
│
▼
[Claude Skill — 接收 csv，呼叫 pipeline.py]
│
▼
[Python — src/pipeline.py，一支跑完]
├── 1. Data Profiling → Issue Log（清洗前的問題記錄）
├── 2. Fixed Cleaning Script（SOP 全部跑一遍）→ Summary Stats（清洗後的狀態）
├── 3. Issue Log + Summary Stats → Gemini API → Validation Summary → Terminal（人工確認）
├── 4. Phase 1: Turnover Analysis
├── 5. Phase 2: CV Segmentation
├── 6. Phase 3: Safety Stock Simulation
├── 7. 分析結果 → Gemini API → 報告文字（Technical Report + Executive Summary）
├── 8. 報告文字 + 三張數據表 → Plotly JSON → Jinja2 注入 HTML 模板 → 輸出 report.html
└── 9. report.html → docs/ → GitHub Pages 部署
```

### 9.3 Tech Stack

| 元件 | 工具 | 職責 |
| --- | --- | --- |
| Exploration | Jupyter Notebook (pandas) | 探索層：profiling 與 EDA 的思考過程；產出清洗規則依據，不參與 production 執行 |
| Data Processing | Python `pipeline.py` (local) | 核心層：資料 profiling（產出 Issue Log）、固定清洗 SOP（產出 Summary Stats）、三階段分析 |
| Cleaning Validation | Gemini API (from Python) | 交叉比對 Issue Log 與 Summary Stats 的合理性，摘要印到 terminal 供人工確認 |
| Report Generation | Gemini API (from Python) | 依不同 prompt 生成 Technical Report 和 Executive Summary |
| Visualization | Plotly (Python `plotly` + Plotly.js CDN) | Python 端建立圖表物件、匯出 Plotly JSON；瀏覽器端以 `Plotly.newPlot()` 渲染互動式圖表（zoom / pan / hover / 匯出） |
| Report Assembly | Jinja2 (local) | 將 Plotly JSON + 報告文字 + 數據表注入 HTML 模板，輸出 report.html |
| Deployment | GitHub Pages (`docs/`) | report.html 放入 `docs/`，啟用 GitHub Pages 產生公開 URL |
| Demo Trigger | Claude Skill | Demo 層：使用者在對話中丟 csv，Skill 呼叫 `pipeline.py` 並回傳 `report.html`；不含分析邏輯 |

### 9.4 Gemini API Responsibilities

| 位置 | Input | Output | 角色 |
| --- | --- | --- | --- |
| 清洗後 | Issue Log + Summary Stats | Validation Summary | 交叉比對清洗前的問題與清洗後的狀態（例如「清洗前 195 筆重複 → 清洗後 0 筆，合理」或「缺失值比例清洗後仍有 3%，需要確認」） |
| 分析後 | Analysis Results JSON | Technical Report | 完整技術報告，含數據細節與圖表說明 |
| 分析後 | Analysis Results JSON | Executive Summary | 給庫存經理的結論與行動建議 |

### 9.5 Quarterly Refresh Design

此 pipeline 設計為每季可重複執行（透過 Claude Skill 或直接呼叫 `pipeline.py`）：

1. 新一季的庫存資料（CSV）到位
2. Python 跑 profiling（產出 Issue Log）→ 固定清洗 SOP（產出 Summary Stats）
3. Issue Log + Summary Stats → Gemini 交叉比對 → 人工確認
4. 確認後跑三階段分析
5. 分析結果 → Gemini API 生成報告文字 → Plotly JSON → Jinja2 組合 → 輸出 `report.html`
6. `report.html` → `docs/` → GitHub Pages 重新部署

---

## 10. Industry Benchmarks

| 項目 | 產業參考值 | 本專案設定 | 來源 |
| --- | --- | --- | --- |
| 庫存周轉率 | 4.5–9 次/年（消費電子） | 正常 SKU 4.5–9，低周轉 < 2 | Onramp Funds, Retalon |
| CV 分群門檻 | CV < 0.2 穩定，CV > 0.7 不穩定 | 0.3 / 0.6（保守設定） | Talcott Ridge, frepple |
| 服務水準 | 分銷商目標 ≥ 95% | 模擬 85% / 90% / 95% | Smart Software, KPI Depot |
| 持有成本率 | 20–30%，電子產品可能 > 30% | 25–30%（依類別） | APICS, Descartes Finale |
| 前置時間（亞洲海運） | 20–35 天（含通關） | 21–35 天（依類別） | Dimerco, Sofeast |
| 髒數據比例 | 企業平均 ~26% | 26% | Experian |

---

## 11. Out of Scope

- 真實資料接入（本專案使用合成資料）
- 即時庫存監控或告警
- 採購訂單自動生成
- 多倉庫 / 多通路分析
- ABC 分析（已在先前專案完成）

---

## 12. Next Steps

1. ~~PRD 定稿~~ ✅
2. ~~從公式反推完整欄位需求，設計新的 Data Dictionary~~ ✅
3. ~~設計 Data Generation Prompt~~ ✅
4. 根據 Generation Prompt 生成合成資料
5. 開發 Python 清洗 + 分析腳本
6. 串接 Gemini API（cleaning validation + 報告生成）
7. 設計 HTML 模板（Jinja2 template + Plotly.js 圖表配置）
8. 端到端測試
9. 部署至 GitHub Pages 並嵌入 README badge
