# Demand-to-Replenishment Engine: Analytical Report

## 1. Executive Summary

I built a weekly forecasting-to-replenishment pipeline for 500 store-item series (10 stores × 50 items, 260 weeks) and tested it in a 52-week out-of-sample inventory simulation. The main results:

- **Forecast accuracy.** Test WAPE is 4.3% for LightGBM, 5.9% for seasonal naive × YoY growth and 13.9% for a 13-week moving average. LightGBM is 27% better than the best simple method and 69% better than the moving average.
- **Inventory at equal service.** For a realized 95% cycle service level with a 1-week lead time, the policy needs 4.4 inventory days with LightGBM, 4.8 with the seasonal benchmark and 6.7 with the moving average. That is 34%–52% fewer days than the moving average (lead time 1 to 3 weeks) and 8%–11% fewer than the seasonal benchmark.
- **Nominal targets mislead.** Setting z for 95% delivered 97.8% realized service with LightGBM but only 85.4% with the moving average.
- **Caveat.** The dataset behaves like generated data (see section 11), so absolute accuracy and inventory-day figures are best-case. The method comparison and the calculation chain are the transferable results.

## 2. Business Context

A retailer replenishing stores weekly has to decide, for every store-item, how much stock covers demand until the next delivery. The order depends on a forecast of demand over the protection interval (lead time plus review period) and on a safety buffer sized to the forecast error. Too little buffer loses sales; too much ties up working capital and, for perishables, creates waste. The S&OP conversation is typically about the trade-off between service level and inventory days. This project quantifies that trade-off from first principles on a public dataset.

## 3. Problem Statement

For a weekly-review replenishment process, how many inventory days are needed to reach a target service level, and how much of that inventory depends on the quality of the demand forecast?

## 4. Objectives and Scope

1. Forecast weekly demand per store-item and compare it with the planner defaults.
2. Convert measured forecast error over the protection interval into safety stock, reorder point and order-up-to level.
3. Simulate the policy on 52 held-out weeks and map service level against inventory days for lead times of 1, 2 and 3 weeks.
4. Test whether ABC-differentiated service targets are worth it.
5. Produce tables that can feed a Power BI S&OP dashboard.

Out of scope: costs and margins (not in the data), promotions, multi-echelon inventory, supplier constraints.

## 5. Dataset Overview

| | |
|---|---|
| Source | Kaggle *Store Item Demand Forecasting Challenge* (playground competition), file mirrored in public GitHub repositories; six copies compared, identical MD5 |
| Content | Daily units sold: date, store, item, sales |
| Size | 913,000 rows; 10 stores × 50 items = 500 series; 2013-01-01 to 2017-12-31 (1,826 days) |
| Not available | Price, cost, promotions, stockouts, lead times, shelf life |

Total units grew from 7.94M in 2013 to 10.73M in 2017 (35.2%), with a summer peak and a weekend lift (Sunday index 1.19, Monday 0.79).

## 6. Data Quality and Preparation

The file passed all checks: no nulls, no duplicate (date, store, item) keys, no negative values, a complete 1,826-day calendar for each of the 500 series, and whole-number units. There is one zero-sales day in 913,000 rows, so demand is not intermittent and no intermittent-demand method was needed. Daily data was aggregated to Monday–Sunday weeks and the incomplete first week (6 days) was dropped, leaving 260 weeks. The raw file is untouched in `data/raw/`.

## 7. Methodology

**Forecast.** One global LightGBM model across all series. Features use data from week *t−4* or earlier (lags 4–8, 13, 26, 51–53; rolling means; year-on-year growth; calendar), scaled by each series' trailing 52-week mean. A single model therefore serves any protection interval up to four weeks. Benchmarks: 13-week moving average (using data up to the previous week), seasonal naive, and seasonal naive × recent YoY growth. Split by time: training, validation (2016-07-04 to 2016-12-26, early stopping and σ estimation), test (2017-01-02 to 2017-12-25, not used for tuning).

**Safety stock.** σ is the standard deviation of forecast error cumulated over every rolling (L+1)-week window of the validation period, per series, instead of assuming a √time rule. Safety stock = z × σ; reorder point = forecast demand over lead time + safety stock; order-up-to level = reorder point + forecast for the review period.

**Simulation.** Weekly review, order-up-to policy, lost sales, 52 test weeks, actual demand. Inventory days = average on-hand ÷ average daily demand. Service is compared at equal *realized* cycle service level by interpolating along each method's z sweep. The simulator has built-in checks (zero-lost-sales with very large z, monotonicity in z and lead time, a perfect-foresight case) after an off-by-one in the order pipeline was caught and fixed during development.

## 8. Key Findings

**Forecast error.** Test WAPE: LightGBM 4.3%, seasonal naive × growth 5.9%, seasonal naive 6.2%, moving average 13.9%. Validation WAPE for LightGBM was 4.4%. Bias on the test year is 1.5% (slight over-forecast). Last year's same-week demand carries 72% of model gain. By ABC class, WAPE is 3.9% (A), 5.5% (B), 6.8% (C).

**Inventory days at 95% realized CSL.**

| Lead time | LightGBM | Seasonal naive × growth | 13-wk moving average | LightGBM vs MA | LightGBM vs seasonal |
|---|---|---|---|---|---|
| 1 week | 4.38 | 4.77 | 6.68 | −34.4% | −8.2% |
| 2 weeks | 4.57 | 5.09 | 8.20 | −44.3% | −10.2% |
| 3 weeks | 4.72 | 5.33 | 9.74 | −51.6% | −11.5% |

The difference sits in the safety buffer. Measured above the z = 0 stock level, the buffer at 95% service is 0.56–0.62 days with LightGBM and 2.9–5.6 days with the moving average.

**Nominal vs realized service (1-week lead time).**

| Nominal target | LightGBM | Seasonal naive × growth | 13-wk moving average |
|---|---|---|---|
| 90% | 95.7% | 91.7% | 75.7% |
| 95% | 97.8% | 95.3% | 85.4% |
| 98% | 99.0% | 97.7% | 93.6% |

**Error scaling.** For LightGBM the empirical interval σ is within 1% of σ₁·√(L+1) at lead times 1–3 (1.01, 1.00, 0.99); weekly errors are nearly uncorrelated (lag-1 autocorrelation 0.02). For the moving average the empirical σ is 1.31–1.63 times the √ rule, so the shortcut would understate the buffer.

**Differentiated service targets (LightGBM, 1-week lead time).** Uniform 95% held 4.62 inventory days at 99.94% fill rate. Tiered A98/B95/C90 held 4.77 days (3.2% more) for a fill rate of 99.96%. Uniform 90% held 4.43 days at 95.7% realized CSL. With 311 A items carrying 80% of volume and every item a fast mover, tiering buys almost nothing here.

**Replenishment parameters, planning week 2018-01-01 (95% nominal).** Safety stock totals 25,518 units for a 1-week lead time (mean 1.3 days of demand per series) and 31,386 units for 2 weeks (1.6 days). Per-series values are in `data/processed/replenishment_parameters_2018W01.csv`.

## 9. Business Implications

The moving average is the weak link. It lags the seasonal ramp, which both inflates forecast error and makes its past errors a poor guide to future ones: its buffer was sized from a validation window that understated what happened later in the year. A seasonal method fixes most of that; machine learning adds a further, smaller improvement. Lead time matters much less once the forecast is good: with LightGBM, going from 1 to 3 weeks adds 0.34 days at 95% service, against 3.06 days with the moving average.

## 10. Recommendations

| # | Recommendation | Priority | Expected outcome (simulated) | Measurement |
|---|---|---|---|---|
| 1 | Replace the moving average with a seasonal forecast (seasonal naive × YoY growth as a minimum; LightGBM where it can be maintained) | High | 34%–52% fewer inventory days vs the MA at equal service | WAPE, bias, forecast value added, monthly |
| 2 | Size safety stock per series from measured σ over lead time + review period, refreshed every cycle | High | Realized service close to target | Realized vs nominal CSL per class |
| 3 | Fix the forecast before funding lead-time reduction; revisit with real supplier lead times | Medium | Avoids paying to shorten lead times a better forecast already neutralises | Inventory days vs lead time by supplier |
| 4 | Keep a uniform service target until item margin or waste cost is available | Low | No loss of service, 3.2% less inventory than the tiered option | Re-test with cost data |
| 5 | Re-run the pipeline on a real extract with promotions and stockout flags before committing to targets | High | Realistic error and buffer sizing | Back-test on ≥52 weeks of real data |

These outcomes are simulation results from one test year; none has been observed in operations.

## 11. Limitations

- **The data looks generated.** After removing the day-of-week effect, total daily demand varies by 0.6% (median CV) inside a month and jumps about 11% at month boundaries, roughly 40× a mid-month change. Public summaries of the competition also note that holiday effects and store closures were pre-processed. Real retail data would be noisier, so accuracy and inventory-day levels here are best-case.
- No cost, price, lead-time, shelf-life or stockout data. Inventory is in days, not money. Lead time is a scenario. Sales are treated as demand.
- Simplified policy: no MOQ, pack sizes, capacity, supplier variability or shelf life; lost sales assumed.
- σ comes from 26 validation weeks (second half of 2016) and is applied to a full year. Realized service departing from nominal is partly a result of this.
- One simulated year without uncertainty bands; the 500 series share one calendar pattern and are not independent.
- The model is not refit during the test year, and a +1.5% test bias slightly flatters LightGBM's service level.

## 12. Conclusion

With 1–3 weeks of lead time, a realized 95% service level needs about 4.4–4.7 inventory days with a seasonal ML forecast, 4.8–5.3 with a seasonal benchmark and 6.7–9.7 with a 13-week moving average. The gap is almost entirely safety buffer and grows with lead time. The calculation chain is reusable; the absolute numbers need re-estimation on real data before they inform decisions.

## 13. References

- Kaggle, *Store Item Demand Forecasting Challenge*. https://www.kaggle.com/c/demand-forecasting-kernels-only (data as mirrored in public GitHub repositories, e.g. https://github.com/tushar2704/Store-Demand-Forecasting).
- St Hakky, *Store Item Demand Forecast Overview* (summary of the competition; notes that holiday effects and store closures were pre-processed). https://book.st-hakky.com/en/data-science/store-item-demand-forecast-overview
- Hyndman, R.J. & Athanasopoulos, G., *Forecasting: Principles and Practice*, 3rd ed. https://otexts.com/fpp3/accuracy.html
- Silver, E.A., Pyke, D.F. & Thomas, D.J., *Inventory and Production Management in Supply Chains*, 4th ed., CRC Press, 2017.
- Ke, G. et al., LightGBM: A Highly Efficient Gradient Boosting Decision Tree, NeurIPS 2017.
