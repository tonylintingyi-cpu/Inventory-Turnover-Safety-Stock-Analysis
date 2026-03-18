# Inventory-Turnover-Safety-Stock-Analysis

# Project Overview:
## Objective:
**Inventory turnover analysis** →Identify SKUs with low turnover ratio that tie up working capital.  

**Demand Volatility Analysis** → For those underperforming SKUs, analyze demand variability using Coefficient of Variation (CV) to segment SKUs into low / medium / high volatility groups, providing the basis for differentiated safety stock strategies.  

**Safety Stock Simulation** → For each volatility segment, simulate different safety stock strategies to find the optimal balance between service level and holding cost.  

**Autonomous workflow** → Build a reusable pipeline that automates data cleaning, validation, and report generation each time new quarterly data arrives — enabling continuous inventory monitoring without manual intervention.  

## Tech stack:

- **Python (Jupyter Notebook):** Conduct data profiling to explore data quality issues such as missing values, outliers, and distribution patterns.
- **Python (Script):** Generate a standalone data cleaning script based on profiling results, ready for automated execution.
- **n8n:** Automate the cleaning pipeline — trigger the Python script and use an AI agent to generate a data validation report (checking data type consistency, missing values, and anomaly detection), also a visualisation report.

# Execution Summary:

# Recommendation: