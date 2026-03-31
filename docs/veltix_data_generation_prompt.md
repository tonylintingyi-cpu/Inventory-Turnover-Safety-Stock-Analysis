# Veltix Dirty Dataset Generation Prompt

## Context

Company: Veltix, a consumer electronics distributor (importing from Asia)
Data Content: Weekly sales and inventory records
Time Range: 2023-W01 ~ 2025-W52 (weekly granularity, 156 weeks)

---

## Product Structure

5 product categories, 10 SKUs each, 50 SKUs total.
15 SKUs are slow-moving items (3 per category), simulating legacy/niche products being over-stocked.

| Product Category | Example Items | Weekly Sales Mean (Normal SKUs) | Unit Cost Range |
|-----------------|---------------|-------------------------------|----------------|
| Mobile Accessories | Phone cases, charging cables, screen protectors | 150–400 | $3–20 |
| Audio | Headphones, Bluetooth speakers | 50–150 | $12–80 |
| Wearables | Smartwatches, fitness bands | 30–80 | $30–140 |
| Computer Peripherals | Keyboards, mice, USB hubs | 40–120 | $9–55 |
| Smart Home | Smart bulbs, plugs, cameras | 20–60 | $15–100 |

---

## Slow-Moving SKU Design (15 SKUs, 3 per category)

**Root Cause**: Low sales volume but excessive inventory.

**Implementation**:
- Weekly sales mean reduced to 25–30% of normal SKUs
- Inventory levels maintained at similar or higher levels than normal SKUs
- Replenishment frequency reduced (every 8–12 weeks vs. 4–6 weeks for normal SKUs)

**Turnover Rate Control**:
- Normal SKUs target annual turnover: 4.5–9x
- Slow-moving SKUs target annual turnover: < 2x
- Annual turnover calculation: (sum of sales_qty / 3) / avg_weekly_inventory
- avg_weekly_inventory = mean of (inventory_begin + inventory_end) / 2 across 156 weeks
- After generation, verify each SKU's annual turnover falls within target range; adjust parameters and re-run if not

---

## Demand Characteristics (Seasonality)

| Category | Seasonality Design | CV Impact on Slow-Moving SKUs |
|----------|-------------------|------------------------------|
| Mobile Accessories | No significant seasonality, low volatility (CV ~ 0.15) | Low CV (< 0.3) |
| Computer Peripherals | Mild seasonality: Q1 back-to-school +20%, Q3 upgrade cycle +20% | Low to Medium CV |
| Smart Home | 10% annual baseline growth, quarterly promo spike +50% | Medium CV |
| Audio | Q4 (W40–W52) surge 2–3x, W45 Singles' Day peak | Medium to High CV |
| Wearables | Q4 (W40–W52) surge 2–3x, W45 Singles' Day peak | High CV (> 0.6) |

Slow-moving SKU CV emerges naturally from seasonality characteristics:
- Categories without seasonality (Mobile Accessories) -> Low CV
- Categories with strong seasonality (Wearables) -> High CV
- No need to artificially inject CV values

---

## Replenishment Logic

**Normal SKUs**:
- Replenish every 4–6 weeks (adjusted by category lead time)
- Order quantity: replenish to target inventory level
- lead_time_days: when receipts > 0, generated from category's normal distribution N(mean, std), rounded to integer, clamped to minimum 7 days
- lead_time_days: when receipts = 0, set to 0 (means no replenishment that week, not zero lead time)

**Slow-Moving SKUs**:
- Replenish every 8–12 weeks
- Order quantity: replenish to a higher target inventory level (maintaining the over-stocked effect)
- lead_time_days: same logic as normal SKUs

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

## Holding Cost Rate (Fixed per Category)

| Category | Annual Holding Cost Rate |
|----------|------------------------|
| Mobile Accessories | 25% |
| Computer Peripherals | 25% |
| Smart Home | 25% |
| Audio | 28% |
| Wearables | 30% |

---

## Column Structure

| Column Name | Description | Generation Logic |
|-------------|-------------|-----------------|
| `sku_id` | SKU identifier (XX-NNN) | Prefix maps to category |
| `product_category` | Product category | Maps to sku_id prefix |
| `week_id` | Week identifier (YYYY-WNN) | 2023-W01 ~ 2025-W52 |
| `sales_qty` | Weekly sales quantity | Based on category mean + seasonality adjustment; slow-moving SKUs reduced to 25–30% |
| `inventory_begin` | Beginning inventory | Current week begin = previous week end |
| `inventory_end` | Ending inventory | = inventory_begin - sales_qty + receipts |
| `receipts` | Weekly replenishment received | Triggered by replenishment cycle; 0 means no replenishment |
| `lead_time_days` | Lead time (days) | Generated from N(mean, std) when receipts > 0; set to 0 when receipts = 0 |
| `unit_cost` | Unit cost (USD) | Fixed per SKU, randomly assigned within category range |
| `holding_cost_rate` | Annual holding cost rate | Fixed per category (0.25–0.30) |

---

## Inventory Equation Constraint

Clean data must satisfy:
```
inventory_end = inventory_begin - sales_qty + receipts
```
And current week's inventory_begin = previous week's inventory_end.

If inventory_begin - sales_qty + receipts < 0, clamp sales_qty so that inventory_end >= 0 (negative inventory is not allowed).

---

## Dirty Data Design (~26% total)

Each type is independently and randomly distributed; overlaps are allowed. Inject sequentially after clean data is fully generated.

| Type | Proportion | Implementation |
|------|-----------|----------------|
| Missing Values | ~7% | Randomly set sales_qty or inventory_begin to NaN, simulating warehouse data entry omissions |
| Duplicate Rows | ~3% | Randomly duplicate entire rows and insert near original position, simulating duplicate system exports |
| SKU Typos | ~4% | Randomly replace or insert characters in sku_id (e.g., MA-001 -> MA-00l, MA_001) |
| Inconsistent Category Names | ~5% | Same category appears in multiple forms (e.g., Audio / audio / AUDIO / Aud. / Audio Products) |
| Leading/Trailing Spaces | ~1.5% | Add leading or trailing whitespace to sku_id or product_category |
| Date Format Inconsistencies | ~2.5% | Mix formats: 2023-W01, 2023-W1, W01-2023, 2023/W01, etc. |
| Negative/Logic Errors | ~2% | Negative sales_qty; or inventory_end != inventory_begin - sales_qty + receipts |
| Outliers | ~1% | sales_qty at 10x normal value or near 0 |

---

## Output Requirements

1. Generate data using Python (pandas + numpy)
2. Build clean data first, ensuring inventory equation holds and turnover rates fall within target ranges
3. After generation, verify: print each SKU's annual turnover rate, confirm normal SKUs are 4.5–9x and slow-moving SKUs < 2x
4. After verification passes, inject dirty data types sequentially
5. Final output as CSV file, filename: `veltix_raw_data.csv`
6. Code must have clear section comments corresponding to each dirty data type
7. At the end of the script, print actual injection counts and proportions for each dirty data type for verification

---

## Constraints

- Do not use external APIs or packages requiring network access
- Random seed set to 42 for reproducibility
- Target total records: 50 SKUs x 156 weeks = 7,800 (actual count slightly higher due to duplicate rows)
