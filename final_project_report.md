# Smart Household Energy Consumption Analytics and Forecasting Using IoT Data
**Final Project Report** - Data Analyst portfolio project (Python, SQL, Machine Learning, Power BI)

## 1. Abstract
This project analyses the UCI *Individual Household Electric Power Consumption* dataset: 2,075,259 one-minute readings from a single household between 16 Dec 2006 and 26 Nov 2010. The data was cleaned with time-aware methods, converted from power (kW) to energy (kWh), aggregated to daily, hourly and monthly levels, explored with Python and SQL (SQLite), and used to build a next-day energy forecasting model. Four models were compared on a chronological 80/20 split. XGBoost achieved the lowest error (MAE 3.82 kWh, RMSE 5.23 kWh, R2 0.49, MAPE 17.5%), about 16% lower MAE than a naive previous-day baseline. Consumption is strongly seasonal, peaks in the evening and is higher at weekends. The work uses historical data only; it is not a real-time IoT system.

## 2. Introduction
Smart meters and sub-meters record electricity use continuously. Turning these records into daily and hourly energy figures helps households and planners understand when and where energy is used. This project uses a public historical dataset to practise the full analytics workflow expected of a Data Analyst: cleaning, exploration, SQL, forecasting and dashboard preparation.

## 3. Problem Statement
Predict the household's **next-day energy consumption (kWh)** using historical consumption patterns, and describe the consumption behaviour that could support demand planning.

## 4. Objectives
1. Clean and validate the raw data without discarding large portions of it.
2. Convert power to energy and aggregate at daily, hourly and monthly level.
3. Discover time-of-day, weekly, seasonal and sub-meter patterns.
4. Answer business questions with SQL.
5. Build and compare forecasting models without data leakage.
6. Prepare Power BI-ready data and dashboard design.

## 5. Dataset Description
| Column | Meaning |
|---|---|
| Date / Time | Timestamp of the reading (one per minute) |
| Global_active_power | Household active power (kW) |
| Global_reactive_power | Household reactive power (kW) |
| Voltage | Voltage (V) |
| Global_intensity | Current intensity (A) |
| Sub_metering_1 | Kitchen active energy (Wh per minute), per UCI documentation |
| Sub_metering_2 | Laundry-room active energy (Wh per minute), per UCI documentation |
| Sub_metering_3 | Water-heater and air-conditioner active energy (Wh per minute), per UCI documentation |

The file has 2,075,259 rows and 9 columns, 1,442 unique dates (1,440 complete days plus a partial first and last day), and no duplicate rows or timestamps. **It is a historical record, not live sensor data.**

## 6. Data Cleaning
* `Date` and `Time` were combined into `DateTime`; the missing-value marker `?` became NaN; all measurements were converted to numeric; chronological order was preserved.
* Validation: no invalid dates, duplicate timestamps, missing timestamps, negative values, zero-power minutes or impossible voltages (223.2-254.2 V). 1,050 minutes had sub-meter energy slightly above total active energy - kept and noted.
* Missing measurements: 25,979 rows (1.25%), always all seven columns together, forming 71 gaps: 64 gaps of up to 3 hours (441 minutes) and 7 longer gaps (25,538 minutes, the longest about 5 days).
* **Short gaps:** time-based interpolation. **Long gaps:** average of the same minute 1-4 weeks earlier (past values only), because a straight line across several days would be unrealistic and using later data could leak future information. All filled minutes are flagged.
* Outliers (94,907 minutes above the IQR fence for active power; maximum 11.12 kW) were kept because they represent real high-load events.
* Daily, hourly and monthly tables use only the 1,440 complete days. 82 days contain some imputed minutes and 21 days have six or more imputed hours ("heavily imputed").

**Energy calculation:** `Energy_kWh = Global_active_power / 60`. Power (kW) is a rate; energy (kWh) is what households use and pay for, and it can be summed over hours, days and months.

## 7. Exploratory Data Analysis
* **Daily:** average 26.13 kWh (median 25.85, std 9.98); maximum 79.56 kWh on 2006-12-23; minimum 4.17 kWh on 2008-08-25; total 37,629.72 kWh over 1,440 days.
* **Monthly / seasonal:** among full calendar months the highest total was Dec 2007 (1,210.09 kWh) and the lowest Aug 2008 (205.74 kWh). Across all years, December averages 35.66 kWh/day and August 13.67 kWh/day; Dec-Feb averages 33.98 vs Jun-Aug 17.39 kWh/day.
* **Hourly:** highest average at 20:00 (1.89 kWh/hour), then 21:00 (1.87) and 19:00 (1.73); lowest at 04:00 (0.44). Weekdays show a morning rise at 07:00 (1.71); weekends rise later and stay higher during the day.
* **Weekday vs weekend:** 24.80 vs 29.46 kWh/day (+18.8%). Saturday (29.80) and Sunday (29.12) are highest; Thursday (23.49) is lowest.
* **Sub-meters:** sub-meter 3 (water heater & air-conditioner) 13,374 kWh (35.5%), sub-meter 2 2,688 kWh (7.1%), sub-meter 1 2,320 kWh (6.2%), not sub-metered 19,247 kWh (51.1%).
* **Distributions:** voltage averages 240.85 V (std 3.23); minute-level active power is right-skewed (median 0.61 kW, mean 1.09 kW).
* **Correlation (daily level):** daily energy correlates very strongly with average current intensity (0.999), and with the not-sub-metered part (0.89), sub-meter 3 (0.73) and peak power (0.72); voltage correlation is weak (0.12). Correlation does not show causation.
* **Trend:** approximately flat - Jan 1-Nov 25 average daily energy was 25.56 (2007), 25.07 (2008), 25.17 (2009), 25.27 (2010) kWh.
* **Unusual days:** 32 days exceed 49.54 kWh (IQR upper fence), all between November and March; no day falls below the lower fence, but the ten lowest days are all in August 2008 (4.17-4.47 kWh). The dataset does not say why (e.g., absence) so no cause is claimed.

## 8. Feature Engineering
Calendar features: Year, Month, Month_Name, Day, DayOfWeek, Day_Name, Week, Quarter, IsWeekend (and Hour in the hourly table). Forecasting features for day *t*: Energy_Today, Lag_1/2/3/7/14/30, Rolling_Mean_3/7/14/30, Rolling_Std_7/30, and calendar features of the target day (day of week, weekend flag, month). Rolling windows end on day *t*. The label is the energy of day *t+1*. A test confirms that altering all later days does not change day *t* features. 31 rows were lost (30 for the longest lag, 1 without a next day), leaving 1,409 rows.

## 9. SQL Analysis
The daily table was loaded into SQLite (`daily_energy`). Queries cover total/average/max/min, monthly totals (`strftime`), top/bottom 10 days, weekday vs weekend (`CASE`), averages by weekday and month, previous-day change (`LAG`), 7-day moving average (window frame), and ranking (`RANK`, also per year with `PARTITION BY`). SQL results agree with the Python results (e.g., total 37,629.72 kWh; average 26.13 kWh; weekend 29.46 vs weekday 24.80 kWh).

## 10. Machine Learning Methodology
* Regression task; target `Next_Day_Energy_kWh`.
* Chronological split, no shuffling: training target days 2007-01-17 to 2010-02-16 (1,127 rows); testing 2010-02-17 to 2010-11-25 (282 rows).
* Models: naive baseline (yesterday's value), Linear Regression (scaled numeric + one-hot day of week and month), Random Forest (300 trees, max depth 8, min leaf 5), XGBoost (300 trees, depth 3, learning rate 0.05). No hyper-parameter search.
* Metrics: MAE, RMSE, R2, MAPE (no classification accuracy).

## 11. Model Evaluation
| Model | MAE | RMSE | R2 | MAPE % |
|---|---|---|---|---|
| Naive baseline | 4.56 | 6.48 | 0.219 | 19.37 |
| Linear Regression | 4.14 | 5.66 | 0.406 | 19.66 |
| Random Forest | 3.96 | 5.34 | 0.471 | 18.73 |
| XGBoost | 3.82 | 5.23 | 0.492 | 17.46 |

XGBoost has the best value on every metric. Compared with the naive baseline its MAE is 16.3% lower and RMSE 19.3% lower. The mean test-day consumption is 23.74 kWh, so the average error is about 16% of a typical day; 74% of test predictions (208 of 282) were within 5 kWh. The mean error is -0.15 kWh (slight over-prediction). Excluding the 11 heavily imputed test days, XGBoost gives MAE 3.83, RMSE 5.27, R2 0.478, MAPE 17.35% - the ranking does not change. Linear Regression has a slightly worse MAPE than the naive baseline despite a better MAE, so the metrics do not all agree for that model. **Feature importance:** rolling means over 3, 7 and 14 days and today's energy were most important (permutation importance highest for Energy_Today and Rolling_Mean_3); this shows what the model relies on, not what causes consumption. The test period (Feb-Nov 2010) does not include a winter, so winter-day accuracy is untested.

## 12. Forecasting
A recursive function predicts one day, appends the prediction to the history and repeats, using no future actuals. XGBoost, refit on all labelled data, forecasts 26 Nov-2 Dec 2010: 28.56, 32.05, 29.29, 26.80, 29.63, 30.82, 28.82 kWh. The forecast stays within a narrow band (26.8-32.0 kWh), typical of recursive forecasts; only one-day-ahead accuracy was evaluated, so the 7-day path is indicative only.

## 13. Power BI Dashboard
Eleven Power BI-ready CSV files were exported. The three-page design (Overview, Patterns, Forecasting) with KPI cards, slicers (Year, Month, Day type), tooltips and 26 DAX measures is documented in `powerbi/README.md` and `powerbi/dax_measures.dax`.

## 14. Business Insights
* Evening (19:00-21:00) is the peak-demand window; night hours (02:00-05:00) are the lowest.
* Winter consumption is roughly twice summer consumption in this household (33.98 vs 17.39 kWh/day).
* Weekends need about 19% more energy per day than weekdays.
* Sub-meter 3 is the largest sub-metered load, yet 51% of energy is not sub-metered, limiting appliance-level insight.
* Demand-planning use: combine recent 3/7-day averages with day-of-week and season to estimate tomorrow's load; monitor days above about 49.5 kWh.

## 15. Limitations
Historical single-household data; not real-time IoT; no weather, occupancy or tariff data; 25,538 minutes filled from earlier weeks; test period excludes winter; moderate forecast accuracy (R2 0.49); recursive 7-day forecast not back-tested; no hyper-parameter tuning; Power BI file not bundled.

## 16. Future Scope (not implemented)
Real-time sensor integration, streaming pipeline, weather and tariff data, anomaly detection, appliance-level forecasting, cloud deployment, automated dashboard refresh, and advanced time-series models with rolling-origin back-testing.

## 17. Conclusion
A complete, reproducible analytics workflow was built on real meter data. It shows clear daily, weekly and seasonal structure and a forecasting model that improves on a naive baseline by about 16% in MAE, while remaining honest about moderate accuracy, imputation and the historical (non-real-time) nature of the data.
