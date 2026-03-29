# Trader Performance vs Market Sentiment (Hyperliquid × Bitcoin Fear/Greed)

This project analyzes how Bitcoin market sentiment (Fear vs Greed) relates to trader behavior and performance on Hyperliquid, and proposes simple, actionable strategy rules based on the findings.[code_file:1]

---

## 1. Project structure

```text
.
├── data/
│   ├── sentiment.csv          # Bitcoin Fear/Greed index (daily)
│   └── trades.csv             # Hyperliquid historical trader data
├── notebooks/
│   └── analysis.ipynb         # Main analysis notebook (Parts A–C)
├── output/
│   ├── data_overview.csv
│   ├── summary_market_by_sent.csv
│   ├── summary_account_by_sent.csv
│   ├── drawdown_quantiles_by_sentiment.csv
│   ├── segment_size_by_sentiment.csv
│   ├── segment_frequency_by_sentiment.csv
│   ├── segment_consistency_by_sentiment.csv
│   ├── avg_pnl_per_account_day_by_sentiment.png
│   ├── avg_trades_per_account_day_by_sentiment.png
│   ├── open_long_short_counts_by_sentiment.png
│   └── segment_size_pnl_by_sentiment.png
└── README.md
```

---

## 2. Setup

### 2.1. Environment

```bash
# Create and activate a virtual environment (optional but recommended)
python -m venv .venv
source .venv/bin/activate      # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

Minimal `requirements.txt`:

```text
pandas
numpy
matplotlib
scikit-learn
```

### 2.2. Data placement

Place the two CSV files in the `data/` folder:

- `data/sentiment.csv` – Bitcoin Fear/Greed history  
  - Columns: `timestamp`, `value`, `classification`, `date`
- `data/trades.csv` – Hyperliquid trader history  
  - Columns like: `Account`, `Coin`, `Execution Price`, `Size Tokens`, `Size USD`,
    `Side`, `Timestamp IST`, `Start Position`, `Direction`, `Closed PnL`, `Trade ID`, …

If your filenames differ, update the paths at the top of the notebook/script.

---

## 3. How to run

### Option A: Jupyter Notebook (recommended)

```bash
jupyter notebook
```

Then open:

- `notebooks/analysis.ipynb`

Run the cells in order:

1. **Data loading & overview**  
   - Loads `sentiment.csv` and `trades.csv`  
   - Documents number of rows/columns, missing values, and duplicates  
   - Parses timestamps and aligns trades with sentiment at daily level  

2. **Feature engineering & key metrics**  
   - Creates:
     - Daily PnL per trader (`pnl_sum`, `pnl_mean`)
     - Win rate per account-day
     - Trade size metrics (`size_usd_sum`, `size_usd_mean`) as a leverage proxy
     - Daily market metrics (trades/day, active accounts/day)
     - Long/short bias metrics (open-long vs open-short counts, ratio)

3. **Analysis (Part B)**  
   - Compares performance on Fear vs Greed vs Neutral days  
   - Studies behavior shifts (trade frequency, position sizes, long/short bias)  
   - Builds trader segments:
     - High vs low frequency
     - High vs low size
     - Consistent vs inconsistent (by win-day rate)

4. **Actionable output (Part C)**  
   - Summarizes methodology, key insights, and strategy recommendations  
   - (Optional) Fits a simple predictive model and/or clustering

### Option B: Script

You can also convert the core notebook cells to a script, e.g. `run_analysis.py`, and run:

```bash
python run_analysis.py
```

This will write the aggregated tables and charts into the `output/` directory.

---

## 4. Methodology (short)

**Data preparation**

- Loaded and cleaned two datasets: daily Bitcoin Fear/Greed sentiment and Hyperliquid trade history.  
- Normalized timestamps:
  - Parsed `date` from sentiment and `Timestamp IST` from trades to datetimes.
  - Created a `trade_date` and `date_dt` at daily granularity and merged on this key.[code_file:1]  
- Standardized trade columns to snake_case and added basic flags (`is_win`, `is_loss`).

**Metrics & aggregation**

- **Account-day level (`daily_account`)**:
  - `trades_count`, `pnl_sum`, `pnl_mean`, `win_rate`
  - `size_usd_sum`, `size_usd_mean` as a proxy for exposure/leverage
- **Market-day level (`daily_market`)**:
  - `trades_count`, `accounts_active`, `pnl_sum`, `pnl_mean`
  - Aggregate wins/losses and size metrics per day
- **Drawdown proxy**:
  - Calculated 10th and 25th percentiles of daily account PnL by sentiment as a “typical bad day” measure.[code_file:3]  

**Segmentation**

- Aggregated per-account metrics over the full sample:
  - `avg_trades_per_day`, `avg_daily_size_usd`, `win_day_rate`, `pnl_std`  
- Split traders into simple segments using medians:
  - High vs low frequency, high vs low size, consistent vs inconsistent.

---

## 5. Key insights (backed by charts/tables)

1. **Fear days are high-opportunity and high-risk**  
   - Average PnL per account-day is highest on Fear days, but the 10th-percentile PnL is also most negative, indicating both greater upside and deeper drawdowns compared to Greed days.[code_file:1][code_file:3][chart:9]  

2. **Trader behavior shifts with sentiment**  
   - Trade frequency and number of active accounts are highest on Fear days and lowest on Greed days, suggesting traders are most active when sentiment is fearful and volatility is elevated.[code_file:2][chart:10]  
   - Long/short positioning flips: traders are net-long on Fear/Neutral days (more open-long than open-short), but tilt net-short on Greed days, indicating a contrarian bias during euphoric sentiment.[code_file:5][chart:11]  

3. **Aggressive segments capture most of the Fear-day edge**  
   - High-frequency and high-size traders earn substantially higher average PnL per account-day on Fear days than low-frequency/low-size traders, while conservative segments benefit less from the elevated volatility.[code_file:8][chart:12]  

These insights are supported by the aggregate tables in `output/summary_market_by_sent.csv`, `output/summary_account_by_sent.csv`, and segment-level summaries in `output/segment_*.csv`.[code_file:2][code_file:6]

---

## 6. Strategy recommendations (rules of thumb)

1. **High-frequency, high-size traders: lean in on Fear, but cap risk**

   - When `sentiment_group == "Fear"`:
     - It can be optimal for aggressive traders to maintain or slightly increase trade frequency and position size, as Fear days show the highest average PnL.
     - However, the worse 10th-percentile PnL on Fear days calls for **strict risk controls**: tighter daily loss limits, smaller incremental adds, and disciplined stop-losses to manage the fatter left tail.[code_file:3][code_file:8]  
   - When `sentiment_group == "Greed"`:
     - Reduce maximum position size and leverage, as the incremental PnL uplift is weaker and does not justify extreme risk-taking.

2. **Low-frequency, low-size traders: focus on Greed/Neutral, de-risk Fear**

   - For more conservative traders:
     - Concentrate trading on **Greed and Neutral** days, where average PnL is decent and drawdowns are milder.
     - On Fear days, **limit trade count and notional** or avoid taking new positions unless the setup is very clear, to avoid being caught in volatile, tail-heavy conditions without enough edge.[code_file:1][code_file:7]  

These rules can be viewed as simple policy templates that condition exposure and aggressiveness on sentiment regime and trader profile.

---

## 7. Optional extensions

The notebook includes (or can easily be extended with):

- **Simple predictive model**  
  - Uses lagged account-day features (previous day’s PnL, trades, win rate, size) and sentiment one-hot encodings to predict whether a trader is profitable the next day.  
  - Implemented with a logistic regression baseline and evaluated on held-out data.

- **Trader archetype clustering**  
  - Runs KMeans on per-account features (`avg_trades_per_day`, `avg_daily_size_usd`, `win_day_rate`, `pnl_std`) to identify behavioral clusters such as “high-frequency volatile”, “casual low-intensity”, and “high-size consistent” traders.

- **Lightweight Streamlit dashboard**  
  - Interactive filters for sentiment and date range.  
  - Displays daily market metrics, per-sentiment PnL, and basic charts to explore the relationship between sentiment and trader behavior.

These extensions are kept intentionally simple and exploratory, focusing on interpretability rather than production-grade models.

---
