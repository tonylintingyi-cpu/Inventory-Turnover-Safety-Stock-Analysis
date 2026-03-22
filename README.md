# Inventory Turnover Safety Stock Analysis

# Project Overview:
## Objective:
**Inventory turnover analysis** →Identify SKUs with low turnover ratio that tie up working capital.  

**Demand Volatility Analysis** → For those underperforming SKUs, analyze demand variability using Coefficient of Variation (CV) to segment SKUs into low / medium / high volatility groups, providing the basis for differentiated safety stock strategies.  

**Safety Stock Simulation** → For each volatility segment, simulate different safety stock strategies to find the optimal balance between service level and holding cost.  

**Autonomous workflow** → Build a reusable pipeline that automates data cleaning, validation, and visualisation dashboard generation each time new quarterly data arrives — enabling continuous inventory monitoring without manual intervention.  

## Tech Stack:

- **Python (local):** Run data profiling, cleaning, and analysis. After cleaning, call Gemini API to generate a validation summary and send it to the user via terminal for manual approval. Once confirmed, POST the analysis results (JSON) to the n8n webhook.  

- **n8n (PikaPods):** Receive analysis results via webhook, then split into two parallel paths —   
    1. Call Gemini API to generate a written report. 
    2. Fetch the pre-built HTML/Chart.js template from GitHub and inject the data to produce an interactive dashboard.   

    Merge both outputs and deliver via email.  

# Data Structure Overview:

# Execution Summary:

# Recommendation: