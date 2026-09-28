# Retail Inventory & Supply Chain Analytics (SQL + Python + Power BI)

An end-to-end supply chain analytics project for a multi-region retail
chain: why are stores running out of stock, which suppliers are actually
causing it, how much is spoilage costing the business, and can a simple
forecast improve reordering? Built using **SQL**, **Python (pandas)**, and
designed to plug directly into **Power BI** for an interactive dashboard.

## Problem Statement

The chain has stockouts eating into sales and spoilage eating into margin,
but no clear view of *why* — is it demand spikes, slow suppliers, unreliable
suppliers, or badly-set reorder points? This project traces stockouts and
spoilage back to their actual drivers across 16 stores, 30 products, and
10 suppliers over a full year of weekly inventory data.

## Tech Stack

- **Python**: pandas, numpy, matplotlib — cleaning, KPI calculation, root
  cause analysis, and a simple demand forecast
- **SQL**: SQL (queries portable to PostgreSQL/MySQL with minor tweaks)
- **Power BI**: a ready-to-import flat dataset + full DAX measures + a
  page-by-page build guide (`powerbi/POWER_BI_GUIDE.md`) — not a `.pbix`
  file, since that's a binary format unsuited to a Git repo, but everything
  needed to build the dashboard yourself in ~20 minutes
- **Data**: synthetic weekly inventory ledger (52 weeks, 16 stores, 30
  products across 5 categories, 10 suppliers) simulating realistic
  reorder-point behavior, supplier lead times, delivery delays, and
  perishable spoilage — intentionally includes missing values, inconsistent
  text, and duplicates for cleaning practice

## Project Structure

```
retail-supply-chain-analytics/
├── data/
│   ├── inventory_ledger.csv         # raw weekly stock ledger (with data quality issues)
│   ├── inventory_ledger_clean.csv   # cleaned data (output of analysis.py)
│   ├── stores.csv                    # store master data
│   ├── products.csv                  # product catalog
│   ├── suppliers.csv                 # supplier master data
│   ├── powerbi_dataset.csv          # flat, joined table ready for Power BI import
│   └── summary_metrics.csv          # key headline metrics
├── sql/
│   └── schema_and_queries.sql       # table schema + 10 business-question queries
├── images/                          # generated charts
├── powerbi/
│   └── POWER_BI_GUIDE.md            # step-by-step dashboard build guide + DAX measures
├── analysis.py                      # cleaning + KPI + root cause + forecast analysis
├── SUPPLY_CHAIN_RECOMMENDATIONS.md  # actionable supplier/inventory/spoilage strategy
└── README.md
```

## How to Run

```bash
pip install pandas numpy matplotlib
python analysis.py
```

This cleans the data, prints KPI and root-cause analysis to the console,
regenerates all charts in `images/`, and exports the Power BI-ready dataset.

To run the SQL queries, load the cleaned CSVs into SQL:
```bash
SQL3 data/supply_chain.db
.mode csv
.import data/inventory_ledger_clean.csv inventory_ledger
.import data/stores.csv stores
.import data/products.csv products
.import data/suppliers.csv suppliers
.read sql/schema_and_queries.sql
```

To build the dashboard, follow `powerbi/POWER_BI_GUIDE.md`.

## Data Cleaning Steps

- Removed exact duplicate rows
- Standardized inconsistent category naming (`packaged foods` / ` Packaged Foods ` → `Packaged Foods`)
- Filled missing `units_received` values with 0 (no delivery recorded that week)

## Key Findings

- **Overall fill rate: 98.2%** (1.8% stockout rate) — solid on average, but
  masks real gaps by category and supplier
- **Supplier reliability drives stockouts more than lead time does**: lead
  time actually correlates *negatively* with stockout rate (-0.45) because
  reorder points already compensate for slow suppliers — but reliability
  correlates as expected (-0.44), meaning **unreliable suppliers, not slow
  ones, are the real risk**
- **Packaged Foods has the highest stockout rate (2.2%)** despite not being
  perishable — a reorder-point sizing issue more than a supply problem
- **Spoilage is heavily concentrated in Perishables**, by far the largest
  share of total spoilage cost — a targeted fix (smaller, more frequent
  orders) rather than a catalog-wide policy change
- **A simple 4-week moving-average forecast hit ~7% MAPE** on the
  highest-volume product — accurate enough to meaningfully improve reorder
  timing without needing a complex model

### Stockout Rate by Category
<img width="1200" height="750" alt="stockout_rate_by_category" src="https://github.com/user-attachments/assets/dd0623cc-cbcf-4035-9f66-7463973441cd" />

### Stockout Rate by Region
<img width="1050" height="750" alt="stockout_rate_by_region" src="https://github.com/user-attachments/assets/494b2738-e8ef-452e-b242-503b01ea1bfa" />

### Inventory Turnover by Category
<img width="1200" height="750" alt="inventory_turnover_by_category" src="https://github.com/user-attachments/assets/e0f1cb85-f7e4-4806-b11a-51675971d55a" />

### Supplier Lead Time vs. Stockout Rate
<img width="1050" height="750" alt="supplier_lead_time_vs_stockouts" src="https://github.com/user-attachments/assets/3f8cf4cd-5820-46cc-b4f7-8e031300f142" />

### Spoilage Cost by Category
<img width="1050" height="750" alt="spoilage_cost_by_category" src="https://github.com/user-attachments/assets/4c1140a1-eb38-4ad9-ad23-56d9b153173c" />

## Demand Forecast vs. Actual

A simple 4-week moving-average forecast was tested against actual weekly
demand for the highest-volume product in the catalog:

<img width="1500" height="750" alt="demand_forecast_vs_actual" src="https://github.com/user-attachments/assets/dd2ef8a5-9316-4a61-a68e-636a1817b6dc" />

**Interpretation:** even this simple approach tracks actual demand closely
(~7% MAPE), suggesting a low-maintenance forecasting layer could meaningfully
tighten reorder timing chain-wide, especially ahead of seasonal demand spikes.

## Supply Chain Recommendations

See [SUPPLY_CHAIN_RECOMMENDATIONS.md](https://github.com/user-attachments/files/32751618/SUPPLY_CHAIN_RECOMMENDATIONS.md) for the full set of supplier, category, and regional recommendations — including an honest note on why the lead-time finding looks backwards at first glance.

## Power BI Dashboard# Supply Chain Recommendations

Actionable moves derived from the stockout, supplier, spoilage, and
forecasting analysis in `[analysis.py](https://github.com/user-attachments/files/32751635/analysis.py)
`.

## 1. Fix the Real Driver: Supplier Reliability, Not Lead Time
"""
Retail Inventory & Supply Chain Analytics
-------------------------------------------
This script:
1. Loads raw weekly inventory ledger + store/product/supplier reference data
2. Cleans it (duplicates, inconsistent text, missing values)
3. Computes core supply chain KPIs: fill rate, stockout rate, inventory
   turnover, days of inventory, spoilage cost, lost sales value
4. Analyzes root causes: stockouts by category/store/region, supplier
   lead-time & reliability vs. stockout rate
5. Builds a simple demand forecast (moving average) vs. actual for one
   product to demonstrate forecasting capability
6. Generates charts
7. Exports a Power-BI-ready clean dataset
"""

import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

pd.set_option("display.float_format", lambda x: f"{x:,.2f}")

# ---------- 1. LOAD ----------
ledger = pd.read_csv("data/inventory_ledger.csv", parse_dates=["week_start"])
stores = pd.read_csv("data/stores.csv")
products = pd.read_csv("data/products.csv")
suppliers = pd.read_csv("data/suppliers.csv")
print(f"Raw inventory_ledger rows: {len(ledger)}")

# ---------- 2. CLEAN ----------
before = len(ledger)
ledger = ledger.drop_duplicates()
print(f"Removed {before - len(ledger)} duplicate rows")

ledger["category"] = ledger["category"].str.strip().str.title()
ledger["units_received"] = ledger["units_received"].fillna(0)

ledger.to_csv("data/inventory_ledger_clean.csv", index=False)
print("Saved cleaned data to data/inventory_ledger_clean.csv")

# join reference data
df = (ledger
      .merge(stores, on="store_id", how="left")
      .merge(products, on="product_id", how="left", suffixes=("", "_prod"))
      .merge(suppliers[["supplier_id", "avg_lead_time_days", "reliability_score"]], on="supplier_id", how="left"))
df["category"] = df["category"].fillna(df["category_prod"])

# ---------- 3. CORE KPIs ----------
total_weeks_rows = len(df)
overall_fill_rate = 100 * (1 - df["stockout_flag"].mean())
overall_stockout_rate = 100 * df["stockout_flag"].mean()
df["lost_sales_value"] = df["lost_sales_units"] * df["unit_price"]
total_lost_sales_value = df["lost_sales_value"].sum()

# inventory turnover (approx): annual COGS / avg inventory value, per SKU
# (per store-product first, then averaged within category -- summing COGS
# across many SKUs and dividing by one row's average inventory value would
# badly inflate the ratio, the same pooling trap as the elasticity calc
# in the pricing project)
df["inventory_value"] = df["closing_stock"] * df["unit_cost"]
df["cogs"] = df["units_sold"] * df["unit_cost"]

sku_turnover = df.groupby(["category", "store_id", "product_id"]).apply(
    lambda g: g["cogs"].sum() / g["inventory_value"].mean() if g["inventory_value"].mean() > 0 else np.nan,
    include_groups=False
).rename("turnover").reset_index()

turnover_by_category = sku_turnover.groupby("category")["turnover"].mean().sort_values(ascending=False)
days_of_inventory_by_category = (365 / turnover_by_category).round(1)

print(f"\nOverall fill rate: {overall_fill_rate:.1f}% | Overall stockout rate: {overall_stockout_rate:.1f}%")
print(f"Total estimated lost sales value: {total_lost_sales_value:,.0f}")
print("\n--- Inventory turnover (annualized) by category ---")
print(turnover_by_category.round(2))
print("\n--- Days of inventory outstanding by category ---")
print(days_of_inventory_by_category)

# ---------- 4. ROOT CAUSE: STOCKOUTS ----------
stockout_by_category = df.groupby("category")["stockout_flag"].mean().sort_values(ascending=False) * 100
stockout_by_region = df.groupby("region")["stockout_flag"].mean().sort_values(ascending=False) * 100
stockout_by_store_type = df.groupby("store_type")["stockout_flag"].mean().sort_values(ascending=False) * 100

print("\n--- Stockout rate by category (%) ---")
print(stockout_by_category.round(1))
print("\n--- Stockout rate by region (%) ---")
print(stockout_by_region.round(1))
print("\n--- Stockout rate by store type (%) ---")
print(stockout_by_store_type.round(1))

# supplier performance vs stockouts
supplier_perf = df.groupby("supplier_id").agg(
    avg_lead_time_days=("avg_lead_time_days", "first"),
    reliability_score=("reliability_score", "first"),
    stockout_rate_pct=("stockout_flag", lambda x: 100 * x.mean()),
).round(2).sort_values("stockout_rate_pct", ascending=False)
print("\n--- Supplier performance vs. stockout rate ---")
print(supplier_perf)

lead_time_stockout_corr = supplier_perf["avg_lead_time_days"].corr(supplier_perf["stockout_rate_pct"])
reliability_stockout_corr = supplier_perf["reliability_score"].corr(supplier_perf["stockout_rate_pct"])
print(f"\nCorrelation (lead time vs. stockout rate): {lead_time_stockout_corr:.2f}")
print(f"Correlation (reliability vs. stockout rate): {reliability_stockout_corr:.2f}")

# ---------- 5. SPOILAGE (PERISHABLES) ----------
df["spoilage_cost"] = df["spoilage_units"] * df["unit_cost"]
spoilage_by_category = df.groupby("category")["spoilage_cost"].sum().sort_values(ascending=False)
total_spoilage_cost = df["spoilage_cost"].sum()
print(f"\n--- Total spoilage cost: {total_spoilage_cost:,.0f} ---")
print(spoilage_by_category.round(0))

# ---------- 6. SIMPLE DEMAND FORECAST (moving average) VS ACTUAL ----------
# pick the highest-volume product for a clean illustrative example
top_product_id = df.groupby("product_id")["units_sold"].sum().idxmax()
top_product_name = products.loc[products["product_id"] == top_product_id, "product_name"].iloc[0]
prod_ts = (df[df["product_id"] == top_product_id]
           .groupby("week_start")["units_sold"].sum()
           .sort_index())

window = 4
forecast = prod_ts.rolling(window=window).mean().shift(1)  # forecast using prior 4 weeks, no lookahead
valid = pd.DataFrame({"actual": prod_ts, "forecast": forecast}).dropna()
mape = (np.abs(valid["actual"] - valid["forecast"]) / valid["actual"].replace(0, np.nan)).mean() * 100
print(f"\n--- Demand forecast (4-week moving average) for '{top_product_name}' ---")
print(f"MAPE: {mape:.1f}%")

# ---------- 7. CHARTS ----------
plt.style.use("seaborn-v0_8-whitegrid")

# 7a. Stockout rate by category
fig, ax = plt.subplots(figsize=(8, 5))
ax.barh(stockout_by_category.index, stockout_by_category.values, color="#C73E1D")
ax.set_title("Stockout Rate by Category (%)", fontsize=14, fontweight="bold")
ax.set_xlabel("Stockout Rate (%)")
plt.tight_layout()
plt.savefig("images/stockout_rate_by_category.png", dpi=150)
plt.close()

# 7b. Stockout rate by region
fig, ax = plt.subplots(figsize=(7, 5))
ax.bar(stockout_by_region.index, stockout_by_region.values, color="#2E86AB")
ax.set_title("Stockout Rate by Region (%)", fontsize=14, fontweight="bold")
ax.set_ylabel("Stockout Rate (%)")
plt.tight_layout()
plt.savefig("images/stockout_rate_by_region.png", dpi=150)
plt.close()

# 7c. Inventory turnover by category
fig, ax = plt.subplots(figsize=(8, 5))
ax.barh(turnover_by_category.index, turnover_by_category.values, color="#F18F01")
ax.set_title("Inventory Turnover by Category (annualized)", fontsize=14, fontweight="bold")
ax.set_xlabel("Turnover Ratio (COGS / Avg Inventory Value)")
plt.tight_layout()
plt.savefig("images/inventory_turnover_by_category.png", dpi=150)
plt.close()

# 7d. Supplier lead time vs stockout rate scatter
fig, ax = plt.subplots(figsize=(7, 5))
ax.scatter(supplier_perf["avg_lead_time_days"], supplier_perf["stockout_rate_pct"],
           s=100, color="#A23B72")
for sid, row in supplier_perf.iterrows():
    ax.annotate(sid, (row["avg_lead_time_days"], row["stockout_rate_pct"]),
                fontsize=8, xytext=(4, 4), textcoords="offset points")
ax.set_title("Supplier Lead Time vs. Stockout Rate", fontsize=14, fontweight="bold")
ax.set_xlabel("Avg Lead Time (days)")
ax.set_ylabel("Stockout Rate (%)")
plt.tight_layout()
plt.savefig("images/supplier_lead_time_vs_stockouts.png", dpi=150)
plt.close()

# 7e. Demand forecast vs actual
fig, ax = plt.subplots(figsize=(10, 5))
ax.plot(valid.index, valid["actual"], label="Actual", color="#2E86AB", marker="o", markersize=3)
ax.plot(valid.index, valid["forecast"], label="Forecast (4-wk moving avg)", color="#C73E1D", linestyle="--")
ax.set_title(f"Demand Forecast vs. Actual — {top_product_name}", fontsize=14, fontweight="bold")
ax.set_ylabel("Units Sold (weekly)")
plt.xticks(rotation=45, ha="right")
ax.legend()
plt.tight_layout()
plt.savefig("images/demand_forecast_vs_actual.png", dpi=150)
plt.close()

# 7f. Spoilage cost by category
fig, ax = plt.subplots(figsize=(7, 5))
ax.bar(spoilage_by_category.index, spoilage_by_category.values, color="#5B8C5A")
ax.set_title("Total Spoilage Cost by Category", fontsize=14, fontweight="bold")
ax.set_ylabel("Spoilage Cost")
plt.xticks(rotation=20, ha="right")
plt.tight_layout()
plt.savefig("images/spoilage_cost_by_category.png", dpi=150)
plt.close()

print("\nCharts saved to images/")

# ---------- 8. SAVE SUMMARY ----------
summary = {
    "overall_fill_rate_pct": round(overall_fill_rate, 1),
    "overall_stockout_rate_pct": round(overall_stockout_rate, 1),
    "total_lost_sales_value": round(total_lost_sales_value, 0),
    "total_spoilage_cost": round(total_spoilage_cost, 0),
    "worst_stockout_category": stockout_by_category.idxmax(),
    "worst_stockout_category_rate_pct": round(stockout_by_category.max(), 1),
    "worst_stockout_region": stockout_by_region.idxmax(),
    "lead_time_stockout_corr": round(lead_time_stockout_corr, 2),
    "reliability_stockout_corr": round(reliability_stockout_corr, 2),
    "demand_forecast_example_product": top_product_name,
    "demand_forecast_mape_pct": round(mape, 1),
}
pd.Series(summary).to_csv("data/summary_metrics.csv")
print("\nSummary:", summary)

# ---------- 9. POWER-BI-READY EXPORT ----------
# a single flat, joined table is the easiest starting point for a Power BI model
powerbi_export = df[[
    "store_id", "store_name", "region", "store_type",
    "product_id", "product_name", "category",
    "week_start", "opening_stock", "units_received", "spoilage_units",
    "units_sold", "closing_stock", "stockout_flag", "lost_sales_units",
    "lost_sales_value", "reorder_point", "safety_stock",
    "supplier_id", "avg_lead_time_days", "reliability_score",
    "unit_cost", "unit_price", "inventory_value", "cogs", "spoilage_cost",
]]
powerbi_export.to_csv("data/powerbi_dataset.csv", index=False)
print("Saved Power BI-ready flat dataset to data/powerbi_dataset.csv")

The data shows something worth double-checking your assumptions on: stockout
rate correlates **negatively** with supplier lead time (-0.45) — slower
suppliers aren't causing more stockouts. That's because reorder points are
already sized up for longer lead times, and it's working. The stronger,
more useful signal is **reliability** (-0.44 correlation, in the expected
direction): suppliers with lower on-time delivery rates (e.g. `SUP003` at
85%, `SUP005`/`SUP006` at 83%) drive more stockouts regardless of how fast
they normally are.

**Recommendation:** Don't renegotiate for faster lead times — renegotiate
(or diversify away from) suppliers with the lowest reliability scores.
A dependable 21-day supplier is safer than an unreliable 6-day one once
reorder points account for the lead time.

## 2. Category-Specific Stockout Fixes

- **Packaged Foods** has the highest stockout rate (2.2%) despite not being
  perishable — this looks like a reorder-point sizing issue, not a supply
  constraint. Recommend reviewing reorder points for this category first.
- **Personal Care** has the lowest stockout rate (1.3%) — current policy is
  working well here; use it as the internal benchmark for other categories.

## 3. Spoilage Is Concentrated and Fixable

Perishables account for the large majority of total spoilage cost — by far
the highest of any category. Since spoilage here comes from unsold stock
sitting too long (aging inventory), the fix is tighter ordering, not more
safety stock:

**Recommendation:** For Perishables specifically, shrink the safety-stock
buffer and order more frequently in smaller batches, even if it slightly
raises stockout risk — the spoilage cost is the bigger loss to eliminate for
short-shelf-life goods.

## 4. West Region Needs a Closer Look

West has the joint-highest stockout rate by region. Cross-reference which
suppliers serve West-region stores specifically — if it's dominated by one
or two lower-reliability suppliers, this may be a sourcing/regional
allocation issue rather than a store-operations issue.

## 5. Demand Forecasting Is a Cheap Win

A simple 4-week moving average forecast on the highest-volume product hit a
~7% MAPE (mean absolute percentage error) — quite accurate for something
this simple. This suggests a lightweight, low-maintenance forecasting layer
(no need for a complex ML model) could meaningfully improve reorder timing
across the catalog, especially ahead of the festive-season demand spike.

## Possible Next Steps

- Build a simple safety-stock optimization: safety stock ∝ supplier
  reliability, not a flat percentage of demand as currently modeled
- Extend the moving-average forecast to every SKU and feed it directly into
  the reorder-point calculation instead of a static formula
- Add supplier cost data to weigh "switch suppliers" recommendations against
  the cost of doing so, not just service level
- A/B test smaller, more frequent Perishables orders on a subset of stores
  before rolling out chain-wide

---
*Note: Dataset is synthetically generated for portfolio/demonstration purposes and does not represent a real business.*


See [`powerbi/POWER_BI_GUIDE.md`](powerbi/POWER_BI_GUIDE.md) for a full
walkthrough: DAX measures, page layouts (Executive Overview, Supplier
Scorecard, Inventory Health), and an optional proper star-schema model.

## Possible Next Steps

- Make safety stock proportional to supplier reliability instead of a flat
  percentage of demand
- Extend the moving-average forecast to every SKU and feed it into the
  reorder-point formula directly
- Add supplier cost data to weigh "switch suppliers" tradeoffs against
  service-level gains
- Publish the Power BI dashboard to the Power BI Service and link it here

---
*Note: Dataset is synthetically generated for portfolio/demonstration purposes and does not represent a real business.*
