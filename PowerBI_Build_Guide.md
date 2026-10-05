# Nassau Candy Distributor — Power BI Build Guide
Product Line Profitability & Margin Performance Analysis

---

## 1. Data quality notes (read before importing)

- **Source data is clean**: 10,194 rows, no nulls, no zero/negative Sales or Units.
- **Division is heavily skewed**: Chocolate = 96.6% of rows, Sugar = 0.4%, Other = 3.0%. Always pair totals with % share or per-unit metrics — raw division totals alone will make Sugar/Other look meaningless.
- **Dates were text (`DD-MM-YYYY`)** — already converted to ISO `YYYY-MM-DD` in the files below.
- **Ship Date is unreliable**: every row shows a 904–1,642 day gap between Order Date and Ship Date. This is almost certainly a data-generation artifact, not real shipping lag. Don't build any delivery-time / lead-time visual from it — it isn't part of your KPI list anyway, so just leave `Ship Date` in the model for reference and ignore it in analysis.
- **Division conflict resolved**: the brief's Product↔Factory table lists "Fizzy Lifting Drinks" under Division "Other," but the actual data tags it "Sugar." I used the **actual data** as source of truth in `Dim_Product`.

---

## 2. Files provided

| File | Contents |
|---|---|
| `Nassau_Candy_PowerBI_Data.xlsx` | One workbook, 4 sheets — import this directly |
| → `Fact_Orders` | 10,194 order lines: keys, dates, Sales, Units, Gross Profit, Cost (Product Name/Division removed — join via Dim_Product) |
| → `Dim_Product` | 15 rows: Product ID, Product Name, Division, Factory |
| → `Dim_Factory` | 5 rows: Factory, Latitude, Longitude (for a map visual) |
| → `Dim_Date` | Full calendar table, 2024-01-02 to 2030-06-28, with Year/Quarter/Month/Weekday columns |

---

## 3. Data model (star schema)

```
Dim_Date (Date)  ──1:*──  Fact_Orders (Order Date)
Dim_Product (Product ID) ──1:*──  Fact_Orders (Product ID)
Dim_Product (Factory) ──*:1──  Dim_Factory (Factory)
```

**Steps in Power BI:**
1. Home → Get Data → Excel → select `Nassau_Candy_PowerBI_Data.xlsx` → load all 4 tables.
2. In **Model view**, drag to create relationships:
   - `Dim_Date[Date]` → `Fact_Orders[Order Date]` (one-to-many, single direction)
   - `Dim_Product[Product ID]` → `Fact_Orders[Product ID]` (one-to-many, single direction)
   - `Dim_Factory[Factory]` → `Dim_Product[Factory]` (one-to-many, single direction)
3. Mark `Dim_Date` as a **Date table**: select it → Table tools → Mark as date table → pick `Date` column.
4. Set `Fact_Orders[Order Date]`'s relationship as **active**; if you ever add a second date relationship (e.g. Ship Date), keep it inactive and use `USERELATIONSHIP` in specific measures only.

---

## 4. DAX measures

Create a dedicated measures table first: Model view → New Table → paste `Measures = ROW("dummy", 1)`, then delete the `dummy` column and hide it. Add all measures below to that table.

### Core aggregates
```dax
Total Sales = SUM(Fact_Orders[Sales])

Total Gross Profit = SUM(Fact_Orders[Gross Profit])

Total Cost = SUM(Fact_Orders[Cost])

Total Units = SUM(Fact_Orders[Units])
```

### KPIs from the brief
```dax
Gross Margin % =
DIVIDE([Total Gross Profit], [Total Sales], 0)

Profit per Unit =
DIVIDE([Total Gross Profit], [Total Units], 0)

Revenue Contribution % =
DIVIDE(
    [Total Sales],
    CALCULATE([Total Sales], ALL(Fact_Orders), ALL(Dim_Product)),
    0
)

Profit Contribution % =
DIVIDE(
    [Total Gross Profit],
    CALCULATE([Total Gross Profit], ALL(Fact_Orders), ALL(Dim_Product)),
    0
)
```

### Margin Volatility (variability of margin over time)
```dax
Margin Volatility =
VAR MonthlyMargins =
    ADDCOLUMNS(
        VALUES(Dim_Date[Year-Month]),
        "MM", DIVIDE(
            CALCULATE([Total Gross Profit]),
            CALCULATE([Total Sales])
        )
    )
RETURN
    STDEVX.P(MonthlyMargins, [MM])
```

### Pareto / profit concentration
```dax
Product Sales Rank =
RANKX(ALL(Dim_Product[Product Name]), [Total Sales], , DESC)

Cumulative Sales % =
VAR CurrentRank = [Product Sales Rank]
VAR TotalAllSales =
    CALCULATE([Total Sales], ALL(Dim_Product))
VAR CumulativeSales =
    CALCULATE(
        [Total Sales],
        FILTER(ALL(Dim_Product), [Product Sales Rank] <= CurrentRank)
    )
RETURN
    DIVIDE(CumulativeSales, TotalAllSales, 0)

Cumulative Profit % =
VAR CurrentRank =
    RANKX(ALL(Dim_Product[Product Name]), [Total Gross Profit], , DESC)
VAR TotalAllProfit =
    CALCULATE([Total Gross Profit], ALL(Dim_Product))
VAR CumulativeProfit =
    CALCULATE(
        [Total Gross Profit],
        FILTER(
            ALL(Dim_Product),
            RANKX(ALL(Dim_Product[Product Name]), [Total Gross Profit], , DESC) <= CurrentRank
        )
    )
RETURN
    DIVIDE(CumulativeProfit, TotalAllProfit, 0)

Is Top 80% by Sales =
IF([Cumulative Sales %] <= 0.8, "Yes", "No")
```

### Margin risk flag (for conditional formatting / a matrix)
```dax
Margin Risk Flag =
VAR M = [Gross Margin %]
RETURN
    SWITCH(
        TRUE(),
        M < 0.2, "High Risk (<20%)",
        M < 0.4, "Watch (20–40%)",
        "Healthy (40%+)"
    )
```

### Division comparison helper (revenue vs profit imbalance)
```dax
Division Sales Share % =
DIVIDE(
    [Total Sales],
    CALCULATE([Total Sales], ALL(Dim_Product[Division])),
    0
)

Division Profit Share % =
DIVIDE(
    [Total Gross Profit],
    CALCULATE([Total Gross Profit], ALL(Dim_Product[Division])),
    0
)

Revenue-Profit Imbalance =
[Division Profit Share %] - [Division Sales Share %]
```
*(Positive = division earns a bigger profit share than its revenue share — efficient. Negative = revenue-heavy but profit-light.)*

### For the date slicer / trend visuals
```dax
Sales YoY % =
VAR PriorYear = CALCULATE([Total Sales], SAMEPERIODLASTYEAR(Dim_Date[Date]))
RETURN DIVIDE([Total Sales] - PriorYear, PriorYear, 0)
```

---

## 5. Page-by-page layout guide

### Page 1 — Product Profitability Overview
- **KPI cards** (top strip): Total Sales, Total Gross Profit, Gross Margin %, Profit per Unit.
- **Product margin leaderboard**: horizontal bar chart, Product Name × `Gross Margin %`, sorted descending, color by `Margin Risk Flag`.
- **Profit contribution**: donut or 100%-stacked bar, Product Name × `Profit Contribution %`.
- **Sales vs Margin scatter**: X = Total Sales, Y = Gross Margin %, size = Total Units, one bubble per product — instantly shows "high-sales/low-margin" outliers (Kazookles will sit bottom-right).
- **Slicers**: Division, Date range, Product search (a slicer on Product Name works as search if you enable the search box in slicer settings).

### Page 2 — Division Performance Dashboard
- **KPI cards**: Gross Margin % and Revenue-Profit Imbalance, per division (use a matrix instead of cards if you want all 3 divisions visible at once).
- **Revenue vs Profit comparison**: clustered column chart, Division on axis, Total Sales and Total Gross Profit as two series.
- **Margin distribution by division**: box-and-whisker or a simple column of `Gross Margin %` by Division (note: with 96.6% of rows in Chocolate, consider showing Order Count alongside so the audience knows Sugar/Other are based on very few transactions).
- **Slicer**: Date range, Region.

### Page 3 — Cost vs Margin Diagnostics
- **Cost-sales scatter**: X = Total Cost, Y = Total Sales, one point per product, trend line on. Points far below the trend line (high cost relative to sales) are your repricing/cost-renegotiation candidates.
- **Margin risk table**: table visual — Product Name, Gross Margin %, Profit per Unit, Margin Risk Flag — conditionally formatted (red/yellow/green) on the flag column.
- **Margin threshold slider**: add a **What-If parameter** (Modeling → New Parameter → Numeric range 0–100%), then filter the risk table visual using that parameter against `[Gross Margin %]` via a measure comparison.

### Page 4 — Profit Concentration (Pareto) Analysis
- **Pareto chart**: combo visual — columns = Total Sales per product (sorted descending), line = `Cumulative Sales %`, with a constant line at 80%.
- **Dependency indicator card**: count of products where `Is Top 80% by Sales = "Yes"` vs total product count — shows how concentrated revenue is in a handful of SKUs.
- Repeat the same combo chart for `Cumulative Profit %` to compare revenue concentration vs profit concentration side by side.

### Slicers to add on every page (sync them via View → Sync Slicers)
- Date range (from `Dim_Date[Date]`)
- Division
- Margin threshold (What-If parameter, page 3 primarily but can sync)
- Product search (Product Name slicer with search enabled)

---

## 6. Optional — Factory map
If you want a 5th page: use `Dim_Factory[Latitude/Longitude]` with the Map or ArcGIS visual, bubble size = `Total Sales` (via the Dim_Product → Dim_Factory relationship), to show which factories are carrying the most revenue/margin.
