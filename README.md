# Fraud Detection System

A three-part fraud detection pipeline built in PostgreSQL and Python, progressing from rule-based detection to a machine learning model with automated reporting and email alerts.

## The problem

Rule-based fraud detection catches known patterns but misses emerging ones. ML models catch more, but lack explainability for operational teams. This project combines both: rules for high-confidence, fast flagging — ML to surface anomalies the rules miss — and automated reporting to close the loop.

## What's in this repo

| Notebook | What it does |
|---|---|
| `Fraud Detection - Stored Procedure.ipynb` | Rule-based detection via PostgreSQL stored procedures: velocity checks, high-value transaction flags, location anomaly detection |
| `Fraud Detection - Machine Learning.ipynb` | Random Forest classifier trained on transaction features, predictions written back to PostgreSQL |
| `Fraud Detection - Automating Fraud Reports.ipynb` | Automated fraud report generation + email alerts triggered by a cron job when fraud rate exceeds threshold |

## Approach

**Stage 1 — Rule-based detection (Stored Procedure)**

Three rule types flagging transactions as suspicious:
- **High-value threshold** — transactions exceeding a defined amount ceiling
- **Velocity check** — multiple transactions from the same customer within a short window
- **Location anomaly** — transactions from geographically distant locations within 1 hour (self-join on timestamp + location delta)

Suspicious transactions are written to a `fraud_flags` table and a stored procedure automates the detection run.

**Stage 2 — Machine Learning layer**

A Random Forest classifier trained on engineered transaction features to catch patterns the rules miss. Features include transaction amount, time-of-day, velocity metrics, and location delta. Predictions are stored back in PostgreSQL alongside the rule-based flags.

> Model performance: **[fill in your accuracy / precision / recall / F1 here]**  
> Training/test split: 80/20 · Class imbalance handled via **[SMOTE / class_weight — fill in]**

**Stage 3 — Automated reporting**

A stored procedure calculates fraud rate per reporting period. A cron job runs it on a schedule and fires email alerts when the fraud percentage exceeds 10%, giving operations teams a live signal without manual querying.

## Key findings

> *(Fill these in from your actual notebook results — this is the most important section for a portfolio)*
>
> Example structure:
> - X% of transactions flagged by rule-based detection; Y% confirmed fraudulent on review
> - ML model improved recall by Z% over rules alone, catching [pattern type] the rules missed
> - Location anomaly rule was the highest-precision signal (low false positive rate)
> - Velocity check generated the most flags but also the most false positives — threshold tuning needed

## Setup

**Requirements**
```
Python 3.10+
PostgreSQL 14+
```

**Install dependencies**
```bash
pip install -r requirements.txt
```

**Database setup**

The notebooks create all required tables and schemas. Connect your PostgreSQL instance at the top of each notebook:
```python
DB_URL = "postgresql://user:password@localhost:5432/fraud_db"
```

**Run order**
1. `Fraud Detection - Stored Procedure.ipynb` — sets up tables and rule-based detection
2. `Fraud Detection - Machine Learning.ipynb` — trains model and adds ML predictions
3. `Fraud Detection - Automating Fraud Reports.ipynb` — sets up reporting + alerts

## Requirements

```
psycopg2-binary
sqlalchemy
pandas
scikit-learn
ipython-sql
```

*(Add a `requirements.txt` to the repo root with these pinned versions)*

## What I'd do next

- **Threshold optimisation** — tune the velocity and high-value thresholds using precision/recall trade-off curves rather than fixed values
- **SHAP values** — add feature importance explainability so operations teams understand *why* a transaction was flagged
- **Real-time scoring** — wrap the ML model in a FastAPI endpoint for live transaction scoring instead of batch runs
- **Drift monitoring** — detect when the fraud pattern shifts so the model can be retriggered for retraining

## Related projects

- [Customer Churn Prediction](https://github.com/guiarpi/customer-churn-prediction) — similar ML pipeline applied to retention
- [Data Quality AI Framework](https://github.com/guiarpi/data-quality-ai-framework) — anomaly detection with LLM reasoning layer
