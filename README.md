# Predicting Bike Station Shortages: Capital Bikeshare

**MGSC 661 Group Project, Group 12**
Ayman Bohsali, Justin Brown, Ajevan Nadarajah, Ruilei Ma

Predict which Capital Bikeshare stations will lose the most bikes in the next hour, so operators know where to send rebalancing crews first.

---

## Overview

Bike-share stations constantly fall out of balance. In the morning rush, residential stations empty out as commuters ride downtown, while downtown stations fill up. Operators fix this by moving bikes with trucks (rebalancing), but crews are limited and often react only after a station is already empty.

This project predicts each station's **net bike loss** (departures minus arrivals) in the **next hour** and ranks stations by need, giving operators an early-warning list.

### Hypotheses

**H1:** Adding recent station activity (net flow over the last few hours and at the same hour last week) significantly improves next-hour predictions over a station's historical hourly average.
- Do models using lagged activity outperform a station × weekday × hour average baseline?
- Which features carry the most predictive weight?

**H2:** Context such as weather, season, weekends and station type changes demand patterns, so a model without context will misjudge certain stations or times.
- How do prediction errors vary by station type, season and time of day?
- Does adding weather and calendar context improve the ranking of the stations most in need?

### What makes this different

Most prior work forecasts total system-wide demand or average error across all stations. Operators only need to know which few stations are about to run out, so we evaluate models on **how well they rank the stations losing the most bikes**, not just overall error.

---

## Data

| Source | What we use it for |
|---|---|
| [Capital Bikeshare trip history](https://capitalbikeshare.com/system-data) | One month per season: **Jan, May, Aug, Oct 2026** (~450,000 trips per month, ~800 stations) |
| [GBFS `station_information` feed](https://gbfs.lyft.com/gbfs/2.3/dca-cabi/en/station_information.json) | Station coordinates and capacity (number of docks) |
| [Open-Meteo historical weather API](https://open-meteo.com/en/docs/historical-weather-api) | Hourly temperature, precipitation and wind for Washington, DC |

Trip files columns: `ride_id`, `rideable_type`, `started_at`, `ended_at`, `start_station_name`, `start_station_id`, `end_station_name`, `end_station_id`, `start_lat`, `start_lng`, `end_lat`, `end_lng`, `member_casual`.

### Cleaning decisions
- **Dockless e-bike trips are excluded.** About 23% of trips have no start or end station; all are e-bikes parked outside a dock.
- **Bad durations are removed:** trips of zero or negative length, or longer than 24 hours.
- **Only the 100 busiest stations are modelled** (chosen on training days only). They carry about half of all activity and keep the dataset manageable.
- **Flow, not inventory:** the trip data records bike movements, not how many bikes are docked, so we predict where bikes leave fastest, not exactly when a station empties.

---

## Method

### Target
`target_next_net_loss` = departures − arrivals at a station in the **next hour**. Positive means the station is losing bikes.

### Train / validation / test split
The four months are not consecutive, so each month is split **by day**, the same way. Every season appears in every split. Data is never shuffled.

| Days of each month | Use |
|---|---|
| 1–7 | Used only to build the one-week lag (dropped) |
| 8–21 | Train |
| 22–25 | Validation |
| 26–end | Test |

All lag features are computed **within each station and month**, so no lag reaches across the gap between two months.

### Models and evaluation
| Task | Models | Metrics |
|---|---|---|
| Regression: predict `target_next_net_loss` | Historical-average baseline, Linear Regression, Random Forest, Gradient Boosting | MAE, RMSE, **precision at top 10** |
| Classification: predict high-loss hours | Logistic Regression, Random Forest | Precision, recall, F1, confusion matrix |

**Precision at top 10:** of the 10 stations the model flags in an hour, how many are truly among that hour's 10 biggest losers.

For classification, "high loss" is defined from the **training set only** and frozen before looking at validation or test data. If the 75th percentile of the target is 0 or 1, use the 90th percentile instead.

---

## Early findings

- **Opposite rush-hour patterns:** a residential station (11th & Kenyon St NW) loses about 8 bikes per hour at 8am, while a downtown station (Farragut Square) gains about 13. The pattern reverses in the evening.
- **Weather matters:** at 10–15°C, average hourly trips fall from 277 when dry to 126 when raining.
- **Seasonality:** winter demand is about 40% of the other seasons (6,244 trips per day in January vs. ~16,000 in August and October).

---

## Repository structure

```
.
├── README.md
├── part1_data_preparation.ipynb   # Part 1: load, clean, aggregate, split, explore
├── data/                          # raw monthly trip CSVs (not committed)
│   ├── 202601-capitalbikeshare-tripdata.csv
│   ├── 202605-capitalbikeshare-tripdata.csv
│   ├── 202608-capitalbikeshare-tripdata.csv
│   └── 202610-capitalbikeshare-tripdata.csv
└── output/                        # shared dataset produced by Part 1
    ├── station_hour_train.csv.gz
    ├── station_hour_val.csv.gz
    └── station_hour_test.csv.gz
```

> The raw CSVs are large, so keep them out of Git. Add `data/` to `.gitignore`.

---

## Getting started

1. Download the four monthly files from the [Capital Bikeshare data bucket](https://s3.amazonaws.com/capitalbikeshare-data/index.html), unzip them, and put the CSVs in `data/`.
2. Install the requirements:
   ```bash
   pip install pandas numpy matplotlib
   ```
3. Open `part1_data_preparation.ipynb`, set `DATA_DIR = "data"` in the configuration cell, and run all cells. An internet connection is needed for the weather and station-capacity data.
4. The shared dataset is written to `output/`. Parts 2–4 load it from there:
   ```python
   import pandas as pd
   train = pd.read_csv("output/station_hour_train.csv.gz", parse_dates=["hour", "target_hour"])
   ```

---

## Dataset columns

| Column | Meaning |
|---|---|
| `target_next_net_loss` | **Target.** Departures minus arrivals in the next hour |
| `baseline_hist_avg` | Training-set average of the target for this station, weekday and hour |
| `station_id`, `station_name`, `season` | Station identifiers and season (winter, spring, summer, fall) |
| `hour`, `target_hour` | Current hour and the hour being predicted |
| `departures`, `arrivals` | Trips leaving and arriving in the current hour |
| `net_lag0`–`net_lag3`, `net_sum_3h` | Net loss in the current hour, the previous 1–3 hours, and their 3-hour total |
| `net_lag_1w`, `dep_lag_1w` | Net loss and departures at the target hour one week earlier |
| `casual_share`, `ebike_share` | Share of this hour's departures by casual riders / on e-bikes |
| `hour_of_day`, `day_of_week`, `is_weekend`, `is_holiday`, `month` | Calendar features of the target hour |
| `lat`, `lon`, `capacity` | Station location and number of docks |
| `temp_c`, `precip_mm`, `wind_kmh`, `is_raining` | Weather at the target hour |

---

## References

- Capital Bikeshare. *System Data.* https://capitalbikeshare.com/system-data
- North American Bikeshare & Scootershare Association. *General Bikeshare Feed Specification (GBFS).* https://github.com/NABSA/gbfs
- Fanaee-T, H., & Gama, J. (2014). Event labeling combining ensemble detectors and background knowledge. *Progress in Artificial Intelligence*, 2(2–3), 113–127.
- Li, Y., Zheng, Y., Zhang, H., & Chen, L. (2015). Traffic prediction in a bike-sharing system. *Proceedings of ACM SIGSPATIAL 2015.*
- Chen, L., Zhang, D., Wang, L., et al. (2016). Dynamic cluster-based over-demand prediction in bike sharing systems. *Proceedings of UbiComp 2016.*
- O'Mahony, E., & Shmoys, D. B. (2015). Data analysis and optimization for (Citi)Bike sharing. *Proceedings of AAAI 2015.*
