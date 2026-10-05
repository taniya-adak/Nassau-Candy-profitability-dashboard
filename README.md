# Product Line Profitability & Margin Performance Analysis
### Nassau Candy Distributor — Power BI Dashboard & Research Paper

A product profitability analysis for a candy distributor, built as part of the Unified Mentor Data Analytics Program. The project identifies which products and divisions genuinely drive profit (as opposed to just revenue), flags margin-risk products, and quantifies how concentrated the business's revenue and profit are across its 15-product portfolio.

## Key Findings
- Overall gross margin: **65.9%** ($141,784 in sales, $93,443 in gross profit) across 10,194 orders, Jan 2024–Dec 2025
- The **Chocolate division** drives 92.9% of sales and carries a 67.4% margin
- Just **5 of 15 products** (all Wonka Bar chocolates) account for over 80% of both total revenue and total profit
- **Kazookles** is a clear margin-risk product: real sales volume (371 units) but only a 7.7% gross margin
- **Lickable Wallpaper** has the best profit-per-unit in the portfolio ($10.00) despite modest volume

## What's in this repo
| File | Description |
|---|---|
| `Nassau_Candy_Distributor.csv` | Raw source dataset (10,194 order lines) |
| `Nassau_Candy_PowerBI_Data.xlsx` | Cleaned, model-ready star schema (Fact_Orders, Dim_Product, Dim_Factory, Dim_Date) |
| `Nassau_Candy_Dashboard.pbix` | The full interactive Power BI dashboard (4 pages) |
| `Nassau_Candy_Research_Paper.docx` | Full write-up: methodology, findings, and recommendations |
| `PowerBI_Build_Guide.md` | Data model, DAX measures, and page-by-page build documentation |
| `screenshots/` | Dashboard page screenshots |

## Dashboard Pages
1. **Product Profitability Overview** — margin leaderboard, profit contribution, sales-vs-margin scatter
2. **Division Performance** — revenue vs. profit comparison, margin by division
3. **Cost vs Margin Diagnostics** — cost-vs-sales scatter with trend line, color-coded margin risk table, live margin threshold slider
4. **Profit Concentration (Pareto)** — cumulative sales % and profit % by product, 80/20 concentration analysis

## Tools
Power BI Desktop · DAX · Power Query

## Author
Taniya Adak 
