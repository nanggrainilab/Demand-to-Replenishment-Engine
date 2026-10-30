# Demand-to-Replenishment Engine
 
Weekly demand forecasting per store-item, converted into safety stock, reorder points and an order-up-to policy, then tested in a 52-week out-of-sample inventory simulation to map **service level against inventory days**.
 
Built as a Supply Chain / Data Analyst portfolio project. Python (pandas, LightGBM), Power BI-ready outputs.
 
## Business problem
 
A retailer replenishing stores weekly has to size orders from a demand forecast plus a buffer for forecast error. The S&OP question is what a given service level costs in inventory, and how much of that cost comes from forecast quality.
 
## Objectives
 
1. Forecast weekly demand for 500 store-item series and compare with planner defaults.
2. Turn measured forecast error over the protection interval into safety stock, reorder point and order-up-to level.
3. Simulate the policy on held-out weeks for lead times of 1, 2 and 3 weeks.
4. Test whether ABC-differentiated service targets are worth it.
## Data
 
Kaggle *Store Item Demand Forecasting Challenge*: 913,000 daily rows, 10 stores × 50 items, 2013-01-01 to 2017-12-31 (no price, cost, promotion, lead-time or stockout fields). Competition page: https://www.kaggle.com/c/demand-forecasting-kernels-only. The raw file is not committed; download it with:
 
 
Check the competition's data terms before redistributing the file. The data behaves like generated data (see Limitations), so this project demonstrates the method rather than describing a real retailer.
 
## Method
 
- Daily → weekly (Monday–Sunday) aggregation, with validation checks on the raw file.
- One global LightGBM model; features use data from at least 4 weeks before the target week. Benchmarks: 13-week moving average, seasonal naive, seasonal naive × YoY growth.
- Time-ordered split: train → validation (26 weeks, early stopping and σ estimation) → test (52 weeks, untouched).
- Interval σ measured empirically per series; safety stock = z·σ; reorder point and order-up-to level derived from the forecast.
- Weekly-review, order-up-to simulation with lost sales. Methods compared at equal *realized* service level. The simulator has built-in sanity checks.
## Key results (test year, 1-week lead time unless noted)
 
| | LightGBM | Seasonal naive × growth | 13-wk moving average |
|---|---|---|---|
| Test WAPE | 4.3% | 5.9% | 13.9% |
| Inventory days for 95% realized service | 4.4 | 4.8 | 6.7 |
| Realized service when target is 95% | 97.8% | 95.3% | 85.4% |
 
- At equal service, LightGBM needs 34–52% fewer inventory days than the moving average (lead time 1–3 weeks) and 8–11% fewer than the seasonal benchmark.
- The difference sits in the safety buffer and widens with lead time.
- Nominal service targets can miss badly if the forecast lags seasonality; back-test realized service.
- ABC-tiered service targets cost 3.2% more inventory for +0.02 pt fill rate in this dataset.
## Recommendations
 
Replace the flat moving average with a seasonal forecast; size safety stock per series from measured interval error; fix forecast quality before funding lead-time reduction; keep uniform service targets until cost data exists; re-run on a real extract with promotions and stockout flags before committing to targets. Details in `reports/analytical_report.md`.
 
 
## Limitations
 
- Total daily demand is flat inside each month and steps at month boundaries, which is unlike real retail. Accuracy and inventory-day levels are best-case.
- No cost, price, lead-time, shelf-life or stockout data; inventory is in days, lead time is a scenario, sales are treated as demand.
- Simplified policy (no MOQ, pack size, capacity, supplier variability); one simulated year without uncertainty bands.
- σ estimated from 26 validation weeks; the model is not refit during the test year.
- The Power BI file is specified (`reports/powerbi_dashboard_spec.md`) but not built.
## References
 
- Kaggle Store Item Demand Forecasting Challenge. https://www.kaggle.com/c/demand-forecasting-kernels-only
- Hyndman & Athanasopoulos, *Forecasting: Principles and Practice*, 3rd ed. https://otexts.com/fpp3/accuracy.html
- Silver, Pyke & Thomas, *Inventory and Production Management in Supply Chains*, 4th ed., CRC Press, 2017.
- Ke et al., LightGBM, NeurIPS 2017.
 
