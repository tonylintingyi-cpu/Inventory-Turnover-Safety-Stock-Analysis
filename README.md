## What This Project Does

Veltix is a consumer electronics distributor that imports from Asia. Like many distributors, it has SKUs sitting in the warehouse tying up capital — some because they're genuinely slow sellers, others because their demand is too erratic to stock efficiently. This project identifies those SKUs, diagnoses why they underperform, and simulates smarter safety stock strategies for each.

The entire workflow is designed to run quarterly: drop in new data, and the pipeline handles cleaning, validation, analysis, reporting, and delivery automatically.

## Analysis Approach

The analysis follows three phases, each building on the previous:

**Phase 1 — Inventory Turnover Analysis.** Calculate turnover for all 50 SKUs using total sales divided by weekly average inventory across 156 weeks. SKUs with annual turnover below 4.5 (the industry lower bound) and in the bottom 30% are flagged as low-performers. (Consumer electronics distributors typically turn inventory 4.5–9× per year.)

**Phase 2 — Demand Volatility Segmentation.** For the ~15 low-turnover SKUs, compute the Coefficient of Variation (CV = σ/μ) on weekly sales — including zero-sales weeks, since sporadic demand is exactly the kind of volatility that matters for stocking decisions. SKUs are grouped into low (CV < 0.3), medium (0.3–0.6), and high (> 0.6) volatility segments. These thresholds are conservative; the literature commonly uses 0.2/0.7, but narrower bands ensure each group has enough SKUs for meaningful simulation.

**Phase 3 — Safety Stock Simulation.** For each volatility segment, simulate safety stock levels at 85%, 90%, and 95% service levels. The formula accounts for both demand variability and lead time variability (derived from actual replenishment weeks only). Output is a tradeoff curve showing holding cost vs. service level for each segment.

## Dataset

Synthetic dataset: 50 SKUs × 156 weeks = 7,800 rows, with ~26% dirty data injected. The data is designed so that normal SKUs land at 4.5–9× annual turnover and 15 low-turnover SKUs (3 per category) fall below 2×. CV segmentation emerges naturally from each category's seasonality profile — Wearables with strong Q4 spikes produce high CV, while stable categories like Mobile Accessories produce low CV.

See `veltix_data_dictionary.md` for field definitions and `veltix_data_generation_prompt.md` for generation logic.

## How the Pipeline Works

**Python (local)** handles all data work. First it profiles the raw data and produces an Issue Log documenting every quality problem found. Then a fixed cleaning script runs the full SOP — deduplication, format correction, missing value handling, etc. — regardless of what the Issue Log says. The cleaning script produces a Summary Stats report showing the post-cleaning state of the data.

Both the Issue Log and Summary Stats are sent to Gemini API for cross-referencing: Gemini compares what was wrong before cleaning with what the data looks like after, and flags anything that doesn't add up. The validation summary prints to the terminal for manual approval. Only after confirmation does analysis proceed through the three phases.

After analysis, the same Python script calls Gemini API twice — once for a technical report (full detail, for the analyst) and once for an executive summary (conclusions and action items, for the inventory manager). The report text and three analysis data tables are then written into a pre-designed Excel template via openpyxl. The template's charts are bound to fixed data ranges, so they update automatically when new numbers are injected. The result is a single `.xlsx` file that any stakeholder can open — no browser, no deployment, no dependencies.

## Tech Stack

| Component | Tool | Role |
| --- | --- | --- |
| Synthetic data generation | Claude Code | Generate dataset based on data generation prompt |
| Data processing & analysis | Python | Profiling (Issue Log), fixed cleaning SOP (Summary Stats), three analysis phases |
| Cleaning validation | Gemini API (called from Python) | Cross-reference Issue Log vs Summary Stats, printed to terminal for approval |
| Report generation | Gemini API (called from Python) | Two audience-specific reports from the same analysis results |
| Output | Excel + openpyxl (local) | Report text + data tables written into pre-designed Excel template; charts auto-update from data ranges |

## Execution Summary:

## Recommendation: