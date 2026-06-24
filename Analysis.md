# Victorian Electricity Demand Forecasting

## 1. Problem Statement
Short-term electricity demand forecasting is critical for grid operators like AusNet, Transgrid and Powercor to balance supply and demand, prevent blackouts, and minimise costs. This project builds a 5-minute ahead demand forecasting system for Victoria using real AEMO operational data.

---

## 2. Data
- **Source:** AEMO Aggregated Price and Demand Data (the exact dataset used by Victorian grid operators)
- **Period:** January 2024 – March 2026 (27 months)
- **Resolution:** 5-minute intervals
- **Size:** 234,000+ records
- **Target variable:** TOTALDEMAND (MW)
- **External data:** Melbourne hourly temperature from Open-Meteo API

---

## 3. Exploratory Data Analysis
Key findings from EDA:
- Demand ranges **2,000–10,500 MW**
- Clear **dual daily peak** - morning (7-8am ~5,100 MW) and evening (6-7pm ~6,100 MW)
- Strong **winter seasonality** - June/July peaks (~5,900 MW) driven by heating
- Extreme demand spikes during heatwaves (Jan 2026 ~9,200 MW)
- Temperature showed **bidirectional effect** - both heat and cold drive demand up
- RRP (spot price) showed weak correlation (0.235) with extreme outliers - excluded as feature

---

## 4. Feature Engineering
Since XGBoost has no built-in notion of time, domain knowledge was used to manually create time-aware features:

**Lag Features** - give the model memory of past demand:
| Feature | Meaning |
|---------|---------|
| lag_1 | 5 minutes ago |
| lag_12 | 1 hour ago |
| lag_48 | 4 hours ago |
| lag_288 | 24 hours ago |
| lag_2016 | 1 week ago |

**Rolling Averages** - capture recent trend:
- rolling_mean_1h - smoothed recent demand
- rolling_mean_24h - daily average trend

**Calendar Features:**
- Hour of day, day of week, is_weekend

**External:**
- Melbourne temperature (merged at hourly resolution)

**Correlation analysis** revealed lag_1 (0.998), rolling_mean_1h (0.979) and lag_12 (0.934) as strongest predictors. Minute, month and quarter showed near-zero correlation and were dropped.

---

## 5. Models

**Baseline - Naive**
Simply uses demand from exactly one week ago as the prediction. No learning involved - just a benchmark to beat.
- MAE: 799 MW | MAPE: 17.34%

**Prophet (Facebook)**
Automatically models trend and seasonality. One line to train - no feature engineering needed. However performed poorly at 5-minute resolution as it is designed for daily/weekly data.
- MAE: 1,366 MW | MAPE: 30.43% ❌
- *Worse than naive baseline - not suitable for high frequency data*

**XGBoost - Main Model**
Gradient boosted trees with engineered lag, calendar and temperature features. Trained on 2024-2025, tested on Jan-Mar 2026.
- MAE: **44 MW** | MAPE: **0.97%** ✅
- 94.4% improvement over naive baseline
- Fast training, fully explainable via SHAP

**LSTM - Deep Learning Comparison**
Two-layer LSTM with sequence length of 288 (24 hours of history). Trained for 10 epochs.
- MAE: 47.73 MW | MAPE: 1.06% ✅
- Comparable to XGBoost but slower to train and harder to explain

---

## 6. Final Results

| Model | MAE | MAPE | Verdict |
|-------|-----|------|---------|
| Naive Baseline | 799 MW | 17.34% | ❌ |
| Prophet | 1,366 MW | 30.43% | ❌ |
| LSTM | 47.73 MW | 1.06% | ✅ Good |
| **XGBoost** | **44 MW** | **0.97%** | ✅ Best |

**XGBoost is the recommended model** - marginally better than LSTM, significantly faster, and fully interpretable.

---

## 7. SHAP Explainability
SHAP analysis confirmed:
- **lag_1** dominates - very recent demand is the strongest signal
- **temperature** shows bidirectional impact - both extreme heat and cold push demand up, consistent with Victorian climate
- **hour** captures peak demand periods
- **is_weekend** and **dayofweek** capture behavioural demand reduction on weekends

---

## 8. Assumptions & Limitations

**Assumptions:**
- Melbourne temperature is representative of broader Victorian demand patterns
- Historical patterns from 2024-2026 will generalise to future periods
- 5-minute ahead forecasting only - not suitable for day-ahead forecasting without modification

**Limitations:**
- No public holiday features included - could improve accuracy around Christmas, Easter
- Single weather station (Melbourne) - regional temperature variation not captured
- Extreme events (heatwaves, storms) are underrepresented in 2 years of data
- Model retraining strategy not implemented - performance may degrade over time

**Reliability:**
- MAPE of 0.97% is strong - industry standard is typically 2-3%
- However tested on only ~3 months of data - longer test period would increase confidence
- Residuals are unbiased and centered around zero - no systematic over/under prediction

---

## 9. Future Improvements
- Add **public holidays** as a binary feature
- Include **multiple weather stations** across Victoria (Geelong, Ballarat, Bendigo)
- Implement **day-ahead forecasting** (predict 288 steps ahead instead of 1)
- Add **model retraining pipeline** - retrain monthly with new AEMO data
-  **LightGBM** - similar to XGBoost but faster, often comparable accuracy
-  **Temporal Fusion Transformer** - state of the art for multi-horizon forecasting
- Deploy as an **API using FastAPI** 



> *"Built a short-term Victorian electricity demand forecasting system using 27 months of AEMO 5-minute operational data. Engineered lag, rolling, calendar and temperature features and benchmarked Naive, Prophet, LSTM and XGBoost models - XGBoost achieved 0.97% MAPE and 44 MW MAE, a 94% improvement over the naive baseline. Applied SHAP for model explainability, identifying temperature and recent demand as dominant drivers."*
