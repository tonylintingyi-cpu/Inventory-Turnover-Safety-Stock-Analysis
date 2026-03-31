# Veltix Raw Data — Data Dictionary

## Data Source

- **Company**: Veltix (consumer electronics distributor, importing from Asia)
- **Time Range**: 2023-W01 ~ 2025-W52 (156 weeks)
- **SKU Count**: 50 (5 categories x 10 SKUs)
- **Raw Record Count**: 7,800 (slightly more after duplicate row injection)
- **Data Quality**: Contains ~26% dirty data

---

## Column Definitions

| Column Name | Type | Description | Valid Range | Notes |
|-------------|------|-------------|-------------|-------|
| `sku_id` | string | SKU identifier | Format `XX-NNN` (e.g., MA-001) | Prefix maps to category: MA/AU/WR/CP/SH |
| `product_category` | string | Product category | Mobile Accessories, Audio, Wearables, Computer Peripherals, Smart Home | — |
| `week_id` | string | Week identifier | Format `YYYY-WNN` (e.g., 2023-W01) | — |
| `sales_qty` | int | Weekly sales quantity (units) | 0 ~ 600 (varies by category and season) | Should not be negative |
| `inventory_begin` | int | Beginning inventory (units) | >= 0 | Current week begin = previous week end |
| `inventory_end` | int | Ending inventory (units) | >= 0 | = inventory_begin - sales_qty + receipts |
| `receipts` | int | Weekly replenishment received (units) | >= 0 | 0 means no replenishment that week |
| `lead_time_days` | int | Replenishment lead time (days) | 14 ~ 56 (varies by category) | Only has value when receipts > 0; 0 when receipts = 0 |
| `unit_cost` | float | Unit cost (USD) | $3 ~ $140 (varies by category) | Fixed per SKU |
| `holding_cost_rate` | float | Annual holding cost rate | 0.25 ~ 0.30 | Fixed per category |

---

## Product Category — SKU Prefix Mapping

| Category | Prefix | SKU Range | Normal SKUs | Slow-Moving SKUs |
|----------|--------|-----------|-------------|------------------|
| Mobile Accessories | MA | MA-001 ~ MA-010 | 7 | 3 |
| Audio | AU | AU-001 ~ AU-010 | 7 | 3 |
| Wearables | WR | WR-001 ~ WR-010 | 7 | 3 |
| Computer Peripherals | CP | CP-001 ~ CP-010 | 7 | 3 |
| Smart Home | SH | SH-001 ~ SH-010 | 7 | 3 |

---

## Demand Characteristics (for sales data generation)

| Category | Weekly Sales Mean (Normal SKUs) | Seasonality | Expected CV Tendency (Slow-Moving SKUs) |
|----------|-------------------------------|-------------|----------------------------------------|
| Mobile Accessories | 150–400 | None | Low CV (< 0.3) |
| Computer Peripherals | 40–120 | Q1 back-to-school +20%, Q3 upgrade cycle +20% | Low to Medium CV |
| Smart Home | 20–60 | 10% annual growth, quarterly promo spike +50% | Medium CV |
| Audio | 50–150 | Q4 surge 2–3x, W45 Singles' Day peak | Medium to High CV |
| Wearables | 30–80 | Q4 surge 2–3x, W45 Singles' Day peak | High CV (> 0.6) |

---

## Slow-Moving SKU Design

- **Count**: 15 (3 per category)
- **Root Cause**: Low sales volume but excessive inventory (simulating legacy/niche products being over-stocked)
- **Sales Volume**: Weekly mean reduced to 25–30% of normal SKUs
- **Inventory Level**: Maintained at similar or higher levels compared to normal SKUs
- **Target Turnover**: < 2x per year (normal SKUs target 4.5–9x per year)
- **Replenishment Frequency**: Every 8–12 weeks (normal SKUs every 4–6 weeks)

---

## Replenishment Logic

| Item | Normal SKUs | Slow-Moving SKUs |
|------|-------------|------------------|
| Replenishment Cycle | Every 4–6 weeks | Every 8–12 weeks |
| Order Quantity | Replenish to target inventory level | Replenish to target level (set higher) |
| lead_time_days | Generated from normal distribution when receipts > 0 | Same as normal SKUs |
| lead_time_days | Set to 0 when receipts = 0 (no replenishment, not zero lead time) | Same as normal SKUs |

---

## Lead Time Reference Values (Normal Distribution)

| Category | Mean (days) | Std Dev (days) |
|----------|-------------|----------------|
| Mobile Accessories | 21 | 5 |
| Audio | 28 | 7 |
| Wearables | 35 | 10 |
| Computer Peripherals | 25 | 6 |
| Smart Home | 30 | 8 |

---

## Holding Cost Rate

| Category | Annual Holding Cost Rate | Rationale |
|----------|------------------------|-----------|
| Mobile Accessories | 25% | Commodity items, low depreciation risk |
| Computer Peripherals | 25% | Commodity items |
| Smart Home | 25% | Commodity items |
| Audio | 28% | Moderate technology refresh |
| Wearables | 30% | Fast technology iteration, high depreciation risk |

---

## Turnover Rate Control

Generation must ensure:
- **Normal SKUs (35)**: Annual turnover rate between 4.5–9x
  - Calculation: annual_turnover = (sum of sales_qty / 3) / avg_weekly_inventory
- **Slow-Moving SKUs (15)**: Annual turnover rate < 2x
  - Achieved by reducing sales mean to 25–30% of normal while maintaining high inventory levels

---

## Known Data Quality Issues (Dirty Data)

This is **raw, uncleaned** data with the following known quality issues, affecting ~26% of records:

| Issue Type | Approx. % | Affected Columns | Description |
|-----------|-----------|------------------|-------------|
| Missing Values | ~7% | sales_qty, inventory_begin | NaN values, simulating warehouse data entry omissions |
| Duplicate Rows | ~3% | All columns | Exact duplicate records |
| SKU Typos | ~4% | sku_id | Variants like MA-00l, MA_001, etc. |
| Inconsistent Category Names | ~5% | product_category | Variants like Audio / audio / AUDIO / Aud. |
| Leading/Trailing Spaces | ~1.5% | sku_id, product_category | Leading or trailing whitespace |
| Date Format Inconsistencies | ~2.5% | week_id | Mixed formats: 2023-W1, W01-2023, 2023/W01, etc. |
| Negative/Logic Errors | ~2% | sales_qty, inventory_end | Negative sales or inventory equation violations |
| Outliers | ~1% | sales_qty | Abnormally high (10x) or abnormally low (near 0) |
