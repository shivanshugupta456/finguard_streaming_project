# FinGuard Streaming Fraud Detection

FinGuard is a streaming data project that simulates credit-card transactions, publishes them to Kafka, processes transaction and fraud-watchlist data in Databricks, and exposes curated outputs for monitoring and alerts.

## Architecture

```text
Python transaction simulator
            |
            v
     Confluent Cloud Kafka
            |
            v
 Databricks Bronze -> Silver -> Gold
                         |          |
                         |          +--> Alerts and dashboard
                         +--> Cleaned transaction data

PostgreSQL customer SQL scripts are provided separately.
```

## Repository layout

```text
dashboard/
  FinGuard Fraud Detection Monitoring.lvdash.json
databricks notebooks and pipelines/
  finguard_project/
    01_kafka_streaming_test.py
    02_Setup_Secret_Scope.py
    03_Send_Email.py
    04_Autoloader_test.py
    finguard_streaming/
      alert/
      bronze/
      gold/
      silver/
    finguard_customers_silver_ingestion/
    fraud_watchlist_file_generator/
kafka_producer/
  producer_normal.py
  producer_fraud_card.py
  producer_fraud_transaction.py
  ...supporting generators, configuration, and models
postgres sql/
  customers_historic.sql
  customers_incremental.sql
```

## Databricks catalog

The workspace screenshots show the `finguard` catalog with the following objects. Table names are written as they appear in the workspace; the `transaciton` spelling is retained to match the existing table names.

| Schema | Tables / objects |
| --- | --- |
| `bronze` | `customers`, `fraud_watchlist`, `fraud_watchlist_batch_test`, `transactions`, `transactions_batch_test`, `transactions_streaming_test` |
| `silver` | `customers`, `fraud_watchlist`, `transactions` |
| `gold` | `fraud_card_alert`, `high_value_transactions_alert`, `transaciton_count_by_minute`, `transaciton_count_by_minute_sliding_window` |
| `source` | Volumes: `fraud_watchlist`, `transactions` |

The Bronze layer ingests raw Kafka transaction and watchlist data. The Silver layer parses and validates transaction records. Gold contains alert and transaction-count outputs. The SQL pipeline definitions and notebook paths are under `databricks notebooks and pipelines/finguard_project/`.

### Databricks notebooks

- `01_kafka_streaming_test.py` tests Kafka batch/stream reads and writes test tables.
- `02_Setup_Secret_Scope.py` contains secret-scope setup code.
- `03_Send_Email.py` contains email notification setup.
- `04_Autoloader_test.py` tests file ingestion with Auto Loader.

Databricks notebooks and pipelines require a configured workspace, appropriate permissions, and the secret/volume configuration expected by the code. Review the notebook settings and paths for your workspace before running them.

## Run the Kafka transaction simulator

### Requirements

- Python 3
- A Confluent Cloud Kafka cluster, topic, and API credentials

From a terminal, create and activate a virtual environment, then install the producer dependencies:

```powershell
cd kafka_producer
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

Create a local `.env` file in `kafka_producer/` with your own Kafka settings:

```env
BOOTSTRAP_SERVERS=
API_KEY=
API_SECRET=
TOPIC_NAME=credit_card_transactions
TRANSACTIONS_PER_SECOND=5
FRAUD_PERCENTAGE=0.08
TOTAL_CUSTOMERS=1000
TOTAL_MERCHANTS=200
RANDOM_SEED=42
```

Do not commit real credentials. Start the continuous normal-transaction producer with:

```powershell
python producer_normal.py
```

The fraud producer scripts can be run separately:

```powershell
python producer_fraud_transaction.py
python producer_fraud_card.py
```

The normal producer runs continuously until stopped. Each fraud producer publishes a single transaction per run.

## PostgreSQL scripts

The `postgres sql/` directory contains a historic customer-data script and an incremental insert/update script. Review the SQL and target database before execution.

## Security and data handling

- **Before publishing this repository, remove and rotate any credentials that have ever been embedded in notebooks or other files.** Use Databricks secrets for workspace credentials and a local, untracked `.env` for the Python producer.
- The historic SQL file and generated datasets may contain personal or sensitive-looking data. Review and sanitize them before sharing the repository publicly.
- Never add API keys, passwords, customer records, card numbers, or private workspace details to this README or to a public repository.
