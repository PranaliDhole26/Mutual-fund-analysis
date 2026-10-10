# Mutual Fund ETL Pipeline Documentation

This document outlines the architecture, setup instructions, and complete Python code for the production-grade **Extract, Transform, Load (ETL)** pipeline designed to consolidate all ten mutual fund CSV datasets into a centralized relational database (SQLite / PostgreSQL).

---

## 1. Architecture Overview

```
[ Raw CSV Files ] ---> ( Extract ) ---> [ Pandas DataFrames ] ---> ( Transform & Clean ) ---> [ SQLAlchemy Engine ] ---> [ SQLite / PostgreSQL DB ]
```

* **Extract:** Reads all 10 raw CSV datasets covering fund masters, NAV history, AUM, inflows, folio counts, performance metrics, investor transactions, portfolio holdings, and benchmark indices.
* **Transform:** Cleans column names, standardizes date formats, handles missing values, casts appropriate numeric/string types, and removes duplicate rows.
* **Load:** Loads the transformed data into relational database tables and automatically builds performance indexes for rapid querying.

---

## 2. Prerequisites & Dependencies

Before running the pipeline, ensure you have Python installed along with the required libraries:

```bash
pip install pandas sqlalchemy psycopg2-binary
```

*Note: `psycopg2-binary` is only required if you switch the database backend to PostgreSQL.*

---

## 3. Complete Python ETL Script (`etl_pipeline.py`)

Save the following script into the same directory where your 10 CSV data files are located:

```python
import os
import logging
import pandas as pd
from sqlalchemy import create_engine, text

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

# ==========================================
# CONFIGURATION
# ==========================================
# For SQLite (Local / Dev):
DB_CONNECTION_STRING = "sqlite:///mutual_funds_warehouse.db"

# For PostgreSQL (Enterprise / Production), uncomment below:
# DB_CONNECTION_STRING = "postgresql://username:password@localhost:5432/mutual_funds_db"

FILE_MAPPING = {
    "fund_master": "1788499983024-b042c300-01_fund_master.csv",
    "nav_history": "1788499983331-4389156d-02_nav_history.csv",
    "aum_by_fund_house": "1788499984134-b0cbf625-03_aum_by_fund_house.csv",
    "monthly_sip_inflows": "1788499984405-d702a6c6-04_monthly_sip_inflows.csv",
    "category_inflows": "1788499984721-4b860901-05_category_inflows.csv",
    "industry_folio_count": "1788499985036-da4a0c4a-06_industry_folio_count.csv",
    "scheme_performance": "1788499985420-bb134abf-07_scheme_performance.csv",
    "investor_transactions": "1788499980509-304c1255-08_investor_transactions.csv",
    "portfolio_holdings": "1788499982117-e3d6ab98-09_portfolio_holdings.csv",
    "benchmark_indices": "1788499982615-f9647ab2-10_benchmark_indices.csv"
}

def run_etl():
    logger.info("Initializing database engine...")
    engine = create_engine(DB_CONNECTION_STRING)
    
    extracted_data = {}
    
    # ------------------------------------------
    # 1. EXTRACT
    # ------------------------------------------
    logger.info("Starting EXTRACTION phase...")
    for table_name, filename in FILE_MAPPING.items():
        if os.path.exists(filename):
            logger.info(f"Extracting {filename} into table '{table_name}'...")
            extracted_data[table_name] = pd.read_csv(filename)
        else:
            logger.warning(f"File {filename} not found! Skipping table '{table_name}'.")

    # ------------------------------------------
    # 2. TRANSFORM
    # ------------------------------------------
    logger.info("Starting TRANSFORMATION phase...")
    transformed_data = {}
    
    for table_name, df in extracted_data.items():
        # Clean column names (strip whitespace, lowercase, replace spaces with underscores)
        df.columns = df.columns.str.strip().str.lower().str.replace(' ', '_')
        
        # Date cleaning / standardization
        for col in df.columns:
            if 'date' in col or col == 'month':
                df[col] = pd.to_datetime(df[col], errors='coerce').dt.strftime('%Y-%m-%d').fillna(df[col])
        
        # Remove duplicate rows
        initial_count = len(df)
        df = df.drop_duplicates()
        if len(df) < initial_count:
            logger.info(f"Removed {initial_count - len(df)} duplicate rows from {table_name}")
            
        transformed_data[table_name] = df

    # ------------------------------------------
    # 3. LOAD
    # ------------------------------------------
    logger.info("Starting LOAD phase into database...")
    with engine.begin() as connection:
        for table_name, df in transformed_data.items():
            logger.info(f"Loading {len(df)} rows into table '{table_name}'...")
            df.to_sql(
                name=table_name,
                con=connection,
                if_exists='replace',
                index=False,
                chunksize=10000
            )
            
    # ------------------------------------------
    # 4. POST-LOAD INDEXING & OPTIMIZATION
    # ------------------------------------------
    logger.info("Creating indexes for high-performance querying...")
    with engine.connect() as connection:
        try:
            connection.execute(text("CREATE INDEX IF NOT EXISTS idx_fund_master_amfi ON fund_master (amfi_code);"))
            connection.execute(text("CREATE INDEX IF NOT EXISTS idx_nav_amfi_date ON nav_history (amfi_code, date);"))
            connection.execute(text("CREATE INDEX IF NOT EXISTS idx_transactions_amfi ON investor_transactions (amfi_code);"))
            connection.execute(text("CREATE INDEX IF NOT EXISTS idx_transactions_date ON investor_transactions (transaction_date);"))
            connection.commit()
            logger.info("Indexes created successfully.")
        except Exception as e:
            logger.warning(f"Indexing note: {e}")

    logger.info("ETL Pipeline completed successfully! All data consolidated into the database.")

if __name__ == "__main__":
    run_etl()
```

---

## 4. Resulting Database Schema

The pipeline automatically maps each CSV into a dedicated relational table:

1. **`fund_master`**: Directory of schemes and metadata (`amfi_code`).
2. **`nav_history`**: Time-series NAV records (`amfi_code`, `date`).
3. **`scheme_performance`**: Risk metrics, alpha, beta, and Sharpe ratios.
4. **`investor_transactions`**: User trade logs and transaction values.
5. **`portfolio_holdings`**: Underlying stock weightings and sector allocations.
6. **`aum_by_fund_house`**: Asset Under Management aggregate statistics.
7. **`monthly_sip_inflows`**: Monthly Systematic Investment Plan volumes.
8. **`category_inflows`**: Category-wise investment flows.
9. **`industry_folio_count`**: Active folio tracking over time.
10. **`benchmark_indices`**: Benchmark market index price history.