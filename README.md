# Real-Time Stocks Market Data Pipeline

![Snowflake](https://img.shields.io/badge/Snowflake-29B5E8?logo=snowflake&logoColor=white)
![DBT](https://img.shields.io/badge/dbt-FF694B?logo=dbt&logoColor=white)
![Apache Airflow](https://img.shields.io/badge/Apache%20Airflow-017CEE?logo=apacheairflow&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white)
![Kafka](https://img.shields.io/badge/Apache%20Kafka-231F20?logo=apachekafka&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?logo=powerbi&logoColor=black)
![MinIO](https://img.shields.io/badge/MinIO-C72E49?logo=minio&logoColor=white)
![Finnhub](https://img.shields.io/badge/Finnhub%20API-1DB954?logoColor=white)

---

## 📌 Project Overview
This project demonstrates an **end-to-end real-time data pipeline** using the **Modern Data Stack**.  
Live stock quotes for **AAPL, MSFT, TSLA, GOOGL, and AMZN** are fetched every few seconds from the **Finnhub API**, streamed through **Apache Kafka**, staged in **MinIO** (S3-compatible object storage), loaded into **Snowflake**, transformed across **Bronze → Silver → Gold** layers using **dbt**, and visualized in **Power BI** — all orchestrated by **Apache Airflow** running in Docker.

![Architecture (1)](https://github.com/user-attachments/assets/6b49eb4d-4bf7-473d-9281-50c20b241760)

---

## ⚡ Tech Stack

| Layer | Tool | Purpose |
|---|---|---|
| Ingestion | Finnhub API + Python | Fetch live stock quotes (AAPL, MSFT, TSLA, GOOGL, AMZN) |
| Streaming | Apache Kafka + Zookeeper | Real-time event streaming via `stock-quotes` topic |
| Monitoring | Kafdrop | Kafka topic UI at `localhost:9000` |
| Staging Storage | MinIO | S3-compatible object store (`bronze-transactions` bucket) |
| Orchestration | Apache Airflow | DAG-based scheduling (every 1 min) |
| Data Warehouse | Snowflake | Cloud warehouse for Bronze → Silver → Gold layers |
| Transformation | dbt | SQL models for cleaning and analytics |
| Visualization | Power BI | Direct Query dashboards from Gold layer |
| Containerization | Docker Compose | All services run locally via containers |

---

## ✅ Key Features
- Polls **live stock prices** (not simulated) for 5 major tickers every few seconds via Finnhub API
- Real-time streaming via **Kafka** on the `stock-quotes` topic with Kafdrop for monitoring
- **MinIO** acts as an S3-compatible staging layer (`bronze-transactions` bucket) before Snowflake ingestion
- Medallion architecture (**Bronze → Silver → Gold**) implemented entirely in **dbt + Snowflake**
- **Airflow DAG** (`minio_to_snowflake.py`) automates MinIO → Snowflake loads on a 1-minute schedule
- **Power BI** connects to Snowflake Gold layer via Direct Query for near-real-time dashboards

---

## 🔧 Prerequisites

Before running this project, make sure you have the following:

- **Docker Desktop** installed and running
- **Snowflake account** (free trial works) with a database, schema, and warehouse created
- **Finnhub API key** — free tier at [finnhub.io](https://finnhub.io)
- **Power BI Desktop** (Windows) for dashboard connection
- Python 3.8+ with dependencies from `requirements.txt`:
  ```
  kafka-python
  boto3
  snowflake-connector-python
  dbt-core
  dbt-snowflake
  requests
  ```

---

## 📂 Repository Structure

```text
real-time-stocks-mds/
├── producer/
│   └── producer.py               # Fetches AAPL, MSFT, TSLA, GOOGL, AMZN from Finnhub → Kafka
├── consumer/
│   └── consumer.py               # Reads from Kafka → writes JSON files to MinIO
├── dbt_stocks/
│   └── models/
│       ├── bronze/
│       │   ├── bronze_stg_stock_quotes.sql   # Raw data from Snowflake staging table
│       │   └── sources.yml                   # dbt source definition
│       ├── silver/
│       │   └── silver_clean_stock_quotes.sql # Cleaned & validated quotes
│       └── gold/
│           ├── gold_candlestick.sql           # OHLC-style aggregation per ticker
│           ├── gold_kpi.sql                   # KPI metrics (price, volume, change)
│           └── gold_treechart.sql             # Market cap / price weight for treemap
├── dag/
│   └── minio_to_snowflake.py     # Airflow DAG: MinIO → Snowflake, runs every 1 min
├── docker-compose.yml            # Spins up Kafka, Zookeeper, Kafdrop, MinIO, Airflow, Postgres
├── requirements.txt
└── README.md
```

---

## 🚀 Getting Started

```bash
# 1. Clone the repo
git clone https://github.com/HariPrakash79/real_time_stocks.git
cd real_time_stocks

# 2. Install Python dependencies
pip install -r requirements.txt

# 3. Start all Docker services (Kafka, Zookeeper, Kafdrop, MinIO, Airflow, Postgres)
docker-compose up -d

# 4. Add your Finnhub API key to producer/producer.py
#    Replace: API_KEY="<<YOUR API KEY>>"

# 5. Run the Kafka producer (starts streaming live stock data)
python producer/producer.py

# 6. Run the Kafka consumer (reads from Kafka → writes to MinIO)
python consumer/consumer.py

# 7. Set up Snowflake manually (create DB, schema, warehouse, and staging table)
#    Then configure dbt profile with your Snowflake credentials

# 8. Run dbt models
dbt run --project-dir dbt_stocks

# 9. Access Airflow at http://localhost:8080 and enable the minio_to_snowflake DAG

# 10. Connect Power BI Desktop to Snowflake Gold layer via Direct Query
```

---

## ⚙️ Step-by-Step Implementation

### **1. Kafka + Docker Setup**
- All services are defined in `docker-compose.yml` and run locally via Docker.
- **Kafka** runs on port `9092` (internal) and `29092` (host machine access).
- **Zookeeper** manages Kafka broker coordination on port `2181`.
- **Kafdrop** (Kafka UI) is available at `http://localhost:9000` to monitor topics and messages.
- The producer and consumer both connect to Kafka via `host.docker.internal:29092`.

---

### **2. Live Market Data Producer**
- [`producer/producer.py`](producer/producer.py) polls the **Finnhub `/quote` endpoint** for 5 tickers: `AAPL`, `MSFT`, `TSLA`, `GOOGL`, `AMZN`.
- Each API response is enriched with `symbol` and `fetched_at` (Unix timestamp) fields.
- Messages are serialized to JSON and published to the **`stock-quotes`** Kafka topic.
- Replace `API_KEY="<<YOUR API KEY>>"` with your Finnhub key before running.

---

### **3. Kafka Consumer → MinIO**
- [`consumer/consumer.py`](consumer/consumer.py) subscribes to the `stock-quotes` topic with `auto_offset_reset="earliest"`.
- Consumed messages are written as individual JSON files to the **`bronze-transactions`** bucket in MinIO.
- MinIO runs locally via Docker at `http://localhost:9002`, using `admin / password123` credentials (change in production).
- The bucket is created automatically if it doesn't exist — making the consumer idempotent on startup.

---

### **4. Airflow Orchestration**
- Airflow is initialized via Docker using the commands in `airflow_initial.txt`.
- [`dag/minio_to_snowflake.py`](dag/minio_to_snowflake.py) defines the pipeline DAG:
  - Reads JSON files from the MinIO `bronze-transactions` bucket
  - Loads records into the **Snowflake Bronze staging table**
  - Runs on a **1-minute schedule**
- Access the Airflow UI at `http://localhost:8080` to monitor and trigger runs.

---

### **5. Snowflake Warehouse Setup**
- Manually create the following in your Snowflake account before running dbt:
  - A **database** (e.g., `STOCKS_DB`)
  - A **schema** (e.g., `RAW`)
  - A **warehouse** (e.g., `STOCKS_WH`)
  - A **staging table** that matches the Bronze dbt source definition in `dbt_stocks/models/bronze/sources.yml`
- Configure your dbt profile (`~/.dbt/profiles.yml`) with your Snowflake account, user, password, and warehouse details.

---

### **6. dbt Transformations**
- Configured **dbt project** (`dbt_stocks/`) with Snowflake as the target warehouse.
- Three-layer medallion architecture:
  - [**Bronze**](dbt_stocks/models/bronze/bronze_stg_stock_quotes.sql) → selects raw records from the Snowflake staging table loaded by Airflow
  - [**Silver**](dbt_stocks/models/silver/silver_clean_stock_quotes.sql) → casts types, filters nulls, deduplicates records
  - [**Gold — Candlestick**](dbt_stocks/models/gold/gold_candlestick.sql) → aggregates open/high/low/close per ticker per time window
  - [**Gold — KPI**](dbt_stocks/models/gold/gold_kpi.sql) → latest price, volume, and price change % per ticker
  - [**Gold — Treechart**](dbt_stocks/models/gold/gold_treechart.sql) → relative price weight across tickers for treemap visualization

---

### **7. Power BI Dashboard**
- Connected **Power BI Desktop** to Snowflake using **Direct Query** on the Gold layer.
- Dashboard includes:
  - **Candlestick chart** → OHLC price patterns per ticker
  - **Tree chart** → relative price weight across AAPL, MSFT, TSLA, GOOGL, AMZN
  - **Gauge charts** → current volume and total trade value per ticker
  - **KPI cards** → real-time price, change %, and volume — sortable by ticker

---

## 📊 Final Deliverables
- Fully automated real-time pipeline from **Finnhub API → Kafka → MinIO → Snowflake → Power BI**
- Medallion architecture with **Bronze → Silver → Gold** dbt models in Snowflake
- **Airflow DAG** orchestrating MinIO → Snowflake loads every minute
- **Power BI dashboard** with live insights on 5 major stock tickers

---

**Author**: *Hariprakash Karthikeyan*

**LinkedIn**: [hariprakashkarthikeyan](https://www.linkedin.com/in/hariprakashkarthikeyan/)

**Contact**: [hariprakashsainik@gmail.com](mailto:hariprakashsainik@gmail.com)