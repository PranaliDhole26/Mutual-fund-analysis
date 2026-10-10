# Rolling Alpha & Tracking Error Analysis Report

This document outlines the analytical framework, data join logic, and complete Python implementation for computing **Rolling Alpha** and **Tracking Error** by joining mutual fund NAV time-series data with benchmark index prices.

## 1. Methodology & Quantitative Framework

1. **Database Join & Alignment:**
   * Maps each fund scheme (`amfi_code`) in `fund_master` to its designated benchmark index (e.g., NIFTY 50 TRI, NIFTY Midcap 150 TRI, BSE SmallCap TRI).
   * Joins daily NAV history (`nav_history`) with benchmark closing prices (`benchmark_indices`) on matching transaction dates.

2. **Daily Returns Calculation:**
   * Computes daily percentage returns for both the fund and its benchmark:
     $$R_{fund, t} = \frac{NAV_t}{NAV_{t-1}} - 1$$
     $$R_{bench, t} = \frac{Index_t}{Index_{t-1}} - 1$$
   * Calculates daily excess return: $E_t = R_{fund, t} - R_{bench, t}$

3. **Rolling Tracking Error Calculation:**
   * Tracking error measures the standard deviation of excess returns over a rolling window (typically 252 trading days = 1 year):
     $$Tracking\ Error = \sqrt{\frac{1}{N-1} \sum (E_t - \bar{E})^2} \times \sqrt{252}$$

4. **Rolling Alpha Calculation:**
   * Measures annualized excess return over the benchmark index across a rolling window:
     $$\alpha_{rolling} = \bar{E} \times 252$$

---

## 2. Complete Python Implementation Script

Save this script as `compute_rolling_metrics.py`:

```python
import sqlite3
import pandas as pd
import numpy as np

def compute_rolling_performance():
    # 1. Connect to SQLite database
    conn = sqlite3.connect("mutual_funds_warehouse.db")

    # 2. Extract NAV history and benchmark mapping
    query = """
    SELECT f.amfi_code, f.scheme_name, f.benchmark, n.date, n.nav
    FROM fund_master f
    JOIN nav_history n ON f.amfi_code = n.amfi_code
    """
    df_nav = pd.read_sql(query, conn)
    df_nav['date'] = pd.to_datetime(df_nav['date'])

    # Extract benchmark prices
    df_bench = pd.read_sql("SELECT date, index_name, close_value FROM benchmark_indices", conn)
    df_bench['date'] = pd.to_datetime(df_bench['date'])

    conn.close()

    # 3. Map fund benchmark names to benchmark_indices index_name keys
    benchmark_mapping = {
        "NIFTY 100 TRI": "NIFTY100",
        "NIFTY 50 TRI": "NIFTY50",
        "NIFTY 500 TRI": "NIFTY500",
        "NIFTY Midcap 150 TRI": "NIFTY_MIDCAP150",
        "BSE 250 SmallCap TRI": "BSE_SMALLCAP",
        "CRISIL Liquid Fund AI Index": "CRISIL_LIQUID",
        "CRISIL Dynamic Gilt Index": "CRISIL_GILT"
    }

    df_nav['mapped_index'] = df_nav['benchmark'].map(benchmark_mapping)

    # 4. Merge NAV data with Benchmark data on date and index_name
    df_merged = pd.merge(
        df_nav, 
        df_bench, 
        left_on=['date', 'mapped_index'], 
        right_on=['date', 'index_name'], 
        how='inner'
    )

    df_merged = df_merged.sort_values(['amfi_code', 'date'])

    # 5. Compute Daily Returns
    df_merged['fund_return'] = df_merged.groupby('amfi_code')['nav'].pct_change()
    df_merged['bench_return'] = df_merged.groupby('amfi_code')['close_value'].pct_change()
    df_merged['excess_return'] = df_merged['fund_return'] - df_merged['bench_return']

    # 6. Compute Rolling Metrics (252-day trading window = 1 Year)
    WINDOW = 252

    df_merged['rolling_tracking_error'] = (
        df_merged.groupby('amfi_code')['excess_return']
        .rolling(window=WINDOW, min_periods=30)
        .std()
        .reset_index(level=0, drop=True) * np.sqrt(252)
    )

    df_merged['rolling_alpha'] = (
        df_merged.groupby('amfi_code')['excess_return']
        .rolling(window=WINDOW, min_periods=30)
        .mean()
        .reset_index(level=0, drop=True) * 252
    )

    # 7. Save back to database
    engine = sqlite3.connect("mutual_funds_warehouse.db")
    df_merged.to_sql('scheme_rolling_metrics', engine, if_exists='replace', index=False)
    engine.close()

    print("Rolling Alpha and Tracking Error computed and stored successfully in 'scheme_rolling_metrics' table!")

if __name__ == "__main__":
    compute_rolling_performance()
```

---

## 3. Resulting Table Structure (`scheme_rolling_metrics`)

| Column Name | Data Type | Description |
| :--- | :--- | :--- |
| `date` | Date | Trading day timestamp |
| `amfi_code` | String | Unique fund scheme identifier |
| `scheme_name` | String | Full name of the mutual fund scheme |
| `nav` | Float | Net Asset Value on date |
| `close_value` | Float | Benchmark index closing price |
| `fund_return` | Float | Daily percentage return of the scheme |
| `bench_return` | Float | Daily percentage return of the benchmark |
| `excess_return` | Float | Difference between fund and benchmark returns |
| `rolling_tracking_error` | Float | 1-year annualized standard deviation of excess returns |
| `rolling_alpha` | Float | 1-year annualized rolling excess return |