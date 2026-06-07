# ChargeOpt-AI: Agentic AI-Based Dynamic Tariff Optimization for EV Charging Networks

## Project Overview

ChargeOpt-AI is an agentic AI-based dynamic pricing framework for EV charging networks. The project addresses the limitation of static fixed-rate EV charging tariffs by predicting station-level demand and utilization, recommending dynamic per-kWh tariffs, and monitoring outcomes through a feedback loop.

The system uses real-world EV charging datasets from ACN-Data and UrbanEV/ST-EVCDP to build a unified station-hour analytical dataset. The final objective is to improve charging station revenue, reduce congestion pressure, increase infrastructure efficiency, and support data-driven tariff decisions.

---

## Problem Statement

Static EV charging tariffs do not respond to real-time operational conditions such as peak-hour congestion, off-peak underutilization, station-level demand variation, and changing charger occupancy. This creates inefficiencies in revenue generation, user experience, and grid utilization.

This project builds a self-improving pricing engine that:

- Predicts next-hour station utilization.
- Identifies overloaded and underutilized stations.
- Recommends dynamic tariffs based on predicted utilization.
- Simulates pricing impact on revenue, utilization, and congestion.
- Uses a Monitoring & Learning Agent to evaluate outcomes and update pricing sensitivity.

---

## Datasets Used

### 1. ACN-Data

ACN-Data provides session-level EV charging records from Caltech/JPL workplace charging networks.

Important fields used:

- `connectionTime`
- `disconnectTime`
- `doneChargingTime`
- `kWhDelivered`
- `stationID` / `spaceID`

ACN was used for session-level preprocessing, charging-duration estimation, energy allocation across active hours, and station-hour aggregation.

### 2. UrbanEV / ST-EVCDP

UrbanEV provides high-frequency public charging station data from Shenzhen.

Files used:

- `volume.csv` — energy demand / kWh load
- `occupancy.csv` — busy charger count
- `duration.csv` — charging duration
- `price.csv` — observed price
- `time.csv` — timestamp reconstruction
- `information.csv` — station metadata
- `adj.csv` — station adjacency structure
- `distance.csv` — spatial distance structure
- `stations.csv` — station information

UrbanEV was used as the main forecasting and tariff simulation dataset because it provides dense station-level time-series data.

---

## Final Unified Dataset

Both datasets were converted into a common station-hour format.

| Metric | Value |
|---|---:|
| Total station-hour rows | 239,245 |
| Total charging stations | 301 |
| ACN rows | 61,405 |
| UrbanEV rows | 177,840 |
| ACN stations | 54 |
| UrbanEV stations | 247 |
| ACN date range | 2018-04-25 to 2018-12-16 |
| UrbanEV date range | 2022-06-19 to 2022-07-18 |

---

## Project Pipeline

```text
Raw EV Charging Data
        ↓
Data Cleaning & Timestamp Alignment
        ↓
Station-Hour Aggregation
        ↓
Feature Engineering
        ↓
Exploratory Data Analysis
        ↓
Demand Prediction Agent
        ↓
Dynamic Tariff Pricing Agent
        ↓
Monitoring & Learning Agent
        ↓
Business Metrics & Final Outputs
```

---

## Preprocessing Decisions

1. ACN session-level records were converted into hourly records by allocating charging energy and charging duration across active hours.
2. UrbanEV 5-minute interval matrices were aggregated into hourly station-level records.
3. Both datasets were aligned using a unified schema containing station ID, timestamp, source, site, demand, occupancy, utilization, price, and revenue.
4. Missing values were handled using transparent assumptions.
5. ACN price was not directly observed, so a fixed baseline tariff of ₹15/kWh was used.
6. UrbanEV prices were preserved and converted into an INR scenario scale for common project comparison.

---

## Engineered Features

Key features created include:

- Utilization Rate = Occupancy Count / Station Capacity
- Revenue = Demand kWh × Tariff
- Revenue per Session
- Energy Cost per kWh
- Occupancy Density
- Queue Proxy = max(0, Utilization Rate − 0.8)
- Peak / Shoulder / Off-Peak period labels
- Hour, day-of-week, weekend, and month features
- Cyclic time features using sine and cosine transformations
- Lag features for utilization, demand, occupancy, and price
- Rolling mean, rolling standard deviation, rolling minimum, and rolling maximum features
- Station-level historical averages
- Station-hour and station-day-hour historical averages
- Network-level congestion and demand indicators

---

## Exploratory Data Analysis Findings

EDA showed strong variation in charging behavior across time, stations, and data sources.

Key findings:

- Average demand was highest during late-night and early-morning hours, especially around 01:00, 06:00, and 00:00.
- Average utilization pressure peaked later in the day, mainly between 09:00 and 13:00.
- 48.90% of station-hour records were underutilized.
- 33.83% of station-hour records were normal.
- 17.27% of station-hour records were overloaded.
- Weekends had higher average demand, while weekdays had higher utilization and queue pressure.
- ACN stations showed higher congestion pressure, while UrbanEV stations showed broader underutilization.
- Several ACN workplace chargers were highly overloaded, while several Shenzhen stations remained underutilized throughout the observed period.

These findings justify the use of utilization-aware dynamic pricing instead of a fixed tariff.

---

## Demand Prediction Agent

The Demand Prediction Agent predicts next-hour station utilization.

### Target Variable

```text
target_next_hour_utilization
```

### Models Tested

- Random Forest Regressor
- Extra Trees Regressor
- Hist Gradient Boosting Regressor

### Best Model Results

| Model | MAE | RMSE | R² |
|---|---:|---:|---:|
| Hist Gradient Boosting | 0.021926 | 0.036240 | 0.956705 |
| Random Forest Strong | 0.027737 | 0.044427 | 0.934933 |
| Extra Trees Strong | 0.030716 | 0.047391 | 0.925963 |

The Hist Gradient Boosting model was selected as the final Demand Prediction Agent because it achieved the lowest RMSE and highest R².

### Demand Agent Outputs

- Predicted next-hour utilization
- Congestion probability proxy
- Expected charging load estimate

---

## Dynamic Tariff Pricing Agent

The Tariff Pricing Agent converts predicted utilization into dynamic tariff recommendations.

### Baseline Tariff

```text
₹15/kWh
```

### Pricing Rules

- If predicted utilization > 80%, apply surge pricing.
- If predicted utilization < 30%, apply mild discount pricing.
- Otherwise, keep normal stable pricing.

### Tariff Bounds

```text
Minimum tariff: ₹14.5/kWh
Maximum tariff: ₹25/kWh
```

### Pricing Simulation

Demand response was simulated using a mild short-run elasticity assumption:

```text
Elasticity = -0.05
```

This means price increases slightly reduce demand, while price decreases slightly increase demand.

The pricing simulation evaluates:

- Baseline revenue
- Dynamic revenue
- Revenue gain percentage
- Recommended tariff distribution
- Pricing action distribution
- Adjusted demand kWh
- Simulated utilization after pricing
- Queue proxy before and after pricing
- Off-peak uplift

---

## Monitoring & Learning Agent

The Monitoring & Learning Agent evaluates pricing decisions and updates pricing sensitivity.

### Monitored Metrics

- Predicted utilization
- Actual next-hour utilization
- Prediction error
- Absolute prediction error
- Baseline revenue
- Dynamic revenue
- Queue proxy before pricing
- Queue proxy after pricing
- Customer response rate
- Pricing efficiency score
- Alpha sensitivity adjustment

### Learning Rule

The agent starts with a pricing sensitivity parameter and adjusts it based on actual utilization:

- If actual utilization > 85%, increase pricing sensitivity.
- If actual utilization < 25%, decrease pricing sensitivity.
- Otherwise, keep sensitivity unchanged.

This creates a simple feedback loop:

```text
Prediction → Tariff Recommendation → Simulated Outcome → Monitoring → Sensitivity Update
```

---

## Evaluation Metrics

### Demand Prediction Agent

- MAE
- RMSE
- R² Score

### Tariff Pricing Agent

- Revenue Gain %
- Charger Utilization before and after pricing
- Off-Peak Uplift
- Recommended Tariff
- Surge / Discount / Normal pricing share

### Monitoring & Learning Agent

- Average Prediction Error
- Average Absolute Prediction Error
- Queue Proxy Reduction
- Customer Response Rate
- Pricing Efficiency Score
- Waiting-Time Reduction Proxy

---

## Output Files

The notebook generates the following outputs:

```text
cleaned_station_hour_data.csv
eda_summary.csv
source_summary.csv
daily_demand_trend.csv
peak_shoulder_offpeak_volatility.csv
spatial_network_summary.csv
model_results.csv
predicted_utilization.csv
dynamic_tariff_recommendations.csv
pricing_summary.csv
extra_tariff_metrics.csv
monitoring_agent_log.csv
monitoring_summary.csv
additional_monitoring_metrics.csv
final_business_metrics.csv
assumptions_and_limitations.csv
metadata.json
plots/*.png
```

---

## Key Visualizations

The project generates visualizations for:

- Average demand by hour
- Average utilization by hour
- Weekday vs weekend demand
- Utilization status distribution
- Top overloaded stations
- Top underutilized stations
- Demand heatmap by day and hour
- Daily demand trend
- Peak / shoulder / off-peak volatility
- Pricing action distribution
- Recommended tariff distribution
- Baseline vs dynamic revenue
- Queue proxy before vs after pricing
- Distance distribution for spatial analysis

---

## Assumptions and Limitations

1. ACN price is not observed, so revenue uses a fixed ₹15/kWh baseline.
2. UrbanEV original prices are preserved and converted into an INR scenario scale for project comparison.
3. ACN station capacity is assumed to be 1 per station ID because charger capacity metadata is unavailable in the uploaded ACN file.
4. Queue length is not directly observed, so queue proxy is defined as max(0, utilization rate − 0.8).
5. Waiting-time reduction is approximated through queue proxy reduction.
6. Demand response is simulated using a mild short-run elasticity assumption.
7. Pricing results are scenario simulations and should not be interpreted as causal claims.
8. UrbanEV is used as the main forecasting dataset because it provides dense station-level time-series data.
9. ACN is used for session-level preprocessing and behavioral charging analysis.

---

## How to Run the Project on Kaggle

1. Upload the dataset folder to Kaggle.
2. Set the dataset path:

```python
DATA_DIR = Path("/kaggle/input/datasets/aklduiawrh/soc-dataset")
```

3. Set the output path:

```python
OUTPUT_DIR = Path("/kaggle/working/ev_tariff_outputs")
PLOTS_DIR = OUTPUT_DIR / "plots"
```

4. Run the notebook cells in order:

```text
Setup
File Inspection
ACN Preprocessing
Shenzhen Preprocessing
Dataset Combination
EDA
Feature Engineering
Demand Prediction Agent
Tariff Pricing Agent
Monitoring & Learning Agent
Final Metrics
Metadata Export
```

---

## Tech Stack

- Python
- Pandas
- NumPy
- Matplotlib
- Scikit-learn
- Random Forest
- Extra Trees
- Hist Gradient Boosting
- Kaggle Notebook Environment

---

## Project Conclusion

ChargeOpt-AI demonstrates how EV charging networks can move from static tariffs to data-driven, utilization-aware pricing. The system predicts next-hour station utilization, recommends bounded dynamic tariffs, and monitors revenue, congestion, and pricing efficiency. The final model achieved strong predictive performance with an R² of 0.956705, showing that station-level temporal patterns can be effectively used for tariff optimization.

The project provides a reproducible pipeline for EV charging demand forecasting, congestion-aware pricing, and agentic monitoring-based improvement.
