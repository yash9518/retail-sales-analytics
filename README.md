# E-commerce Sales & Customer Analytics (SQL · Excel · Power BI)

**Business question:** a UK online gift retailer wants to know:
- How much it really sells, once cancellations, fees and data errors are cleaned out.
- When and where its revenue comes from.
- Which customers to keep, which to win back, and which products to bundle.

**Data:** the UCI *Online Retail* dataset: **541,909 transaction lines**, 1 Dec 2010 to 9 Dec 2011, covering 38 countries. Most buyers are small wholesalers. See `data/DATA_LICENSE.md` for source and licence (CC0).

**Approach:**
- **SQL first.** All the business logic is in `sql/`, run on SQLite. It uses CTEs, joins and window functions: `ROW_NUMBER`, `LAG`, `RANK`, `NTILE`, `FIRST_VALUE` and running `SUM OVER`.
- **Python** only orchestrates the pipeline, runs one statistical test and draws the charts.
- **Results** are delivered as an **Excel dashboard** (`excel/`) and a **Power BI star schema with a build guide and DAX measures** (`powerbi/`).

---

## Key findings

| # | Finding | Where |
|---|---------|-------|
| 1 | **Raw data overstates sales by 4.3%.** 2,805 sale lines (£424K) were fully cancelled later, so they are not real revenue. Two of them were bulk orders of **80,995 and 74,215 units (£246K)**, cancelled within 12 and 16 minutes. Another was a **£38,970 price-entry error**: picnic baskets keyed at £649.50 instead of £4.95. | `sql/02_match_cancellations.sql` |
| 2 | **Clean KPIs:** **£9.82M revenue**, 19,648 orders, 4,324 identified customers. The average order is **£500** and the average basket is **272 units**, which confirms these are wholesale buyers. **15.4%** of revenue comes from guest orders with no customer ID. | `04_kpi_summary.sql` |
| 3 | **Strong seasonality.** September to November 2011 brought **36% of all revenue in the period**. November peaked at **£1.44M**, 1.8x an average month. | `05_monthly_trend.sql` |
| 4 | **UK = 84.7% of revenue**, but foreign buyers order bigger baskets. Their median order is **£384 vs £295** for the UK. A Mann-Whitney test gives p < 0.001, and the bootstrapped 95% CI of the gap is £68 to £113. The Netherlands is 2.9% of revenue from only 9 customers. | `06_country_revenue.sql`, Python |
| 5 | **Revenue is concentrated.** The top 20% of customers bring **74%** of revenue, and the top 20% of products bring **78%**. | `07_product_pareto.sql` |
| 6 | **Retention is the weak spot.** Only **~20%** of customers acquired in 2011 bought again the following month (2011 cohort average 19.8%). The December 2010 cohort contains customers already active when the data starts; it held 37% in month 1 and 50% in November 2011. | `08_cohort_retention.sql` |
| 7 | **RFM segments:** **Champions** are 26% of customers but bring **69% of revenue from identified customers**. **649 At-Risk customers** (frequent buyers who have gone quiet for about 5 months on average) account for **£0.8M** of revenue, which makes them the win-back list. | `09_rfm_scores.sql`, `10_rfm_segment_summary.sql` |
| 8 | **Cross-sell pairs:** 7,771 product pairs were mined with a SQL self-join. The strongest are matching sets, such as paper cups with paper plates in the same design (**lift 81**, 88% confidence), so they are natural bundles. | `11_basket_pairs.sql` |
| 9 | **Order timing:** **no orders are ever placed on a Saturday**. 78% of orders arrive between 10:00 and 16:00, peaking at noon. | `12_weekday_hour.sql` |

### Recommendations
1. **Win back the At-Risk segment** (649 customers, £0.8M of revenue) with a targeted offer before the next peak season, and measure it with a holdout group.
2. **Bundle the top cross-sell pairs** on product pages and in wholesale catalogues.
3. **Plan stock and staffing for September to November**, which brings over a third of annual revenue.
4. **Fix data capture.** Make customer ID mandatory (15% of revenue is unattributable), and add order-entry checks for very large quantities and unusual prices.
5. **Treat retention as the headline KPI.** About 80% of new customers do not come back the next month.

---

## Dashboards

**Excel** (`excel/Retail_Sales_Dashboard.xlsx`): pick a country in the yellow cell, and every KPI card, the monthly table and the trend chart recalculate.
- Built with `SUMIFS`, `INDEX`/`MATCH`, data validation and conditional formatting.
- Includes a colour-scaled cohort heatmap on its own sheet.
- Rebuild it with `python src/build_excel_dashboard.py`.

**Power BI** (`powerbi/`):
- **Star schema:** `fact_sales`, `fact_returns`, `dim_customer` (with RFM segment), `dim_product` and `dim_date`.
- **`DAX_measures.md`:** revenue, AOV, MoM growth, running total, return rate, segment share and cohort retention.
- **`BUILD_GUIDE.md`:** builds a 3-page report covering Overview, Customers (with the cohort matrix) and Products, plus a table of target numbers to check your report against SQL.

![Monthly revenue](reports/figures/01_monthly_revenue.png)
![Cohort retention](reports/figures/02_cohort_retention.png)
![RFM segments](reports/figures/03_rfm_segments.png)

---

## SQL pipeline

| File | What it does | SQL techniques |
|---|---|---|
| `01_clean_and_classify.sql` | Removes 5,268 exact duplicates and labels each line as sale, cancellation, non-product (postage, fees, manual adjustments, vouchers) or stock adjustment | `DISTINCT`, `ROW_NUMBER`, `CASE`, `LIKE … ESCAPE` |
| `02_match_cancellations.sql` | Pairs each cancellation with the sale it reverses: same customer, product and quantity, and the latest sale before the cancellation | self-join, `ROW_NUMBER() OVER (PARTITION BY …)` |
| `03_star_schema.sql` | Builds the fact and dimension tables (most common product name, customer's main country) | CTEs, `ROW_NUMBER` |
| `04`–`07` | KPIs, monthly trend with MoM growth and running total, country ranking, product Pareto with return rates | `LAG`, `SUM() OVER`, `RANK`, window-frame cumulative sums |
| `08_cohort_retention.sql` | Monthly acquisition cohorts and their retention | `MIN() OVER (PARTITION BY)`, `FIRST_VALUE` |
| `09`–`10` | RFM quintile scores and segments | `NTILE(5)`, `CASE` |
| `11_basket_pairs.sql` | Support, confidence and lift for product pairs | self-join on invoice, `HAVING` |
| `12`–`13` | Weekday-by-hour demand, and the month x country cube for Excel | `strftime`, `GROUP BY` |

## Assumptions and limitations
- **Revenue definition.** Revenue means product sales in GBP. It excludes postage, fees, manual adjustments, sales that were fully cancelled and the cancellations themselves. Partial returns (£90K, 0.9% of sales) are reported separately.
- **Cancellation matching.** A cancellation is matched on customer, product and quantity. When its price differs from the original sale, it is treated as a correction of that sale; the picnic-basket line is an example.
- **December 2011** only covers 1 to 9 December, so it is excluded from peak-month and retention averages.
- **RFM tie-breaks.** RFM quintiles use `NTILE`, so customers with equal frequency are split by their spend.
- **Single retailer.** This is one UK retailer between 2010 and 2011. The methods transfer to other businesses; the exact numbers do not.

## Run it
```bash
pip install -r requirements.txt
python src/retail_analysis.py        # ~40 s: SQL pipeline, charts, reports/, powerbi/data/
python src/build_excel_dashboard.py  # rebuilds excel/Retail_Sales_Dashboard.xlsx
```
