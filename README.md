# Fraud Detection System — PostgreSQL + Python + Machine Learning

An end-to-end fraud detection pipeline combining rule-based detection, machine learning, and automated reporting — built on 284,807 real transactions from the Kaggle Credit Card Fraud dataset.

---

## The Problem

Credit card fraud costs the global financial industry over $30 billion annually. The core challenge isn't just building a model — it's navigating the **precision-recall trade-off**: flagging too much legitimate activity frustrates customers and creates operational overhead, while missing actual fraud causes direct financial loss.

This project demonstrates three complementary approaches to that problem, each adding a layer the previous one can't handle alone:

| Stage | Approach | Why |
|---|---|---|
| 1 | **Rule-based detection** | Fast, explainable, zero false negatives for known patterns |
| 2 | **Machine learning** | Catches subtle patterns rules can't articulate |
| 3 | **Automated reporting** | Surfaces fraud trends and alerts without manual intervention |

---

## Dataset

**[Kaggle — Credit Card Fraud Detection](https://www.kaggle.com/datasets/mlg-ulb/creditcardfraud)**

- 284,807 transactions from European cardholders (September 2013)
- 492 fraudulent transactions (**0.172%** — severe class imbalance)
- Features: `Time`, `V1–V28` (PCA-anonymised), `Amount`, `Class`

> The dataset is not included in this repo due to file size. See Setup below — `kagglehub` downloads it automatically on first run.

---

## Key Results

| Model | Precision (Fraud) | Recall (Fraud) | ROC-AUC |
|---|---|---|---|
| Logistic Regression (balanced) | ~0.06 | ~0.92 | ~0.97 |
| Random Forest (balanced) | ~0.87 | ~0.82 | ~0.98 |

**Why recall matters more than accuracy here:** a model that predicts "not fraud" for all 284,807 transactions scores 99.83% accuracy — and catches zero fraud. Recall (the share of actual frauds caught) is the primary business metric. The Random Forest trades some recall for much higher precision, meaning far fewer false positives sent to the fraud operations team.

*Exact figures vary by run. Full evaluation in `02-machine-learning.ipynb`.*

---

## Project Structure

```
fraud-detection/
├── 00-eda.ipynb                  # Exploratory analysis & visualisations
├── 01-stored-procedure.ipynb     # Rule-based detection with PostgreSQL stored procedures
├── 02-machine-learning.ipynb     # Logistic Regression vs. Random Forest with full evaluation
├── 03-automated-reports.ipynb    # Fraud reports, alerts, and cron job automation
├── plots/                        # Generated visualisation outputs
├── requirements.txt
└── .gitignore
```

---

## Notebooks Overview

### `00-eda.ipynb` — Exploratory Analysis
- Class imbalance visualisation (577:1 legitimate-to-fraud ratio)
- Fraud rate by hour of day
- Transaction amount distributions by class
- Top discriminating PCA features correlated with fraud
- Correlation heatmap

### `01-stored-procedure.ipynb` — Rule-Based Detection
Three detection rules, each targeting a different fraud pattern:
- **High-value threshold** — transactions exceeding a defined ceiling
- **Velocity check** — multiple transactions from the same customer within a short window
- **Location anomaly** — transactions from geographically distant locations within 1 hour (self-join on timestamp + location delta)

Results written to a `flagged_transactions` table. A stored procedure automates daily execution.

### `02-machine-learning.ipynb` — ML Pipeline
- Class imbalance handling via `class_weight='balanced'`
- Logistic Regression and Random Forest — trained, evaluated, and compared
- Full evaluation: confusion matrix, classification report, ROC curve, Precision-Recall curve
- Feature importance analysis
- Threshold tuning — choosing the operating point on the precision/recall curve based on operational cost of false positives vs. false negatives
- Predictions stored back to PostgreSQL alongside rule-based flags

### `03-automated-reports.ipynb` — Reporting & Alerts
- Self-join SQL to detect suspicious transaction sequences
- Stored procedure for automated fraud rate calculation per reporting period
- Alert logic: triggers when fraud rate exceeds configurable threshold (default 10%)
- Cron job setup for scheduled execution

---

## Setup

### Prerequisites
- Python 3.9+
- PostgreSQL 14+
- Free Kaggle account (for dataset download)

### 1. Kaggle API credentials

```bash
# 1. Go to kaggle.com → profile → Settings → API → Create New Token
# 2. Move the downloaded kaggle.json:
mkdir -p ~/.kaggle
mv ~/Downloads/kaggle.json ~/.kaggle/kaggle.json
chmod 600 ~/.kaggle/kaggle.json
```

`kagglehub` in the notebooks handles the download automatically on first run.

### 2. Install dependencies

```bash
git clone https://github.com/guiarpi/Fraud-Detection.git
cd Fraud-Detection

python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt
```

### 3. Database setup

```bash
psql -U postgres -c "CREATE DATABASE fraud_detection;"
```

Update the connection string at the top of each notebook:

```python
engine = create_engine('postgresql://postgres:YOUR_PASSWORD@localhost:5432/fraud_detection')
```

> Store credentials in a `.env` file and load with `python-dotenv`. Never commit credentials to git.

### 4. Run order

```
00-eda.ipynb → 01-stored-procedure.ipynb → 02-machine-learning.ipynb → 03-automated-reports.ipynb
```

---

## Skills Demonstrated

| Area | Specifics |
|---|---|
| **SQL** | Stored procedures, self-joins, window functions, CTEs, PostgreSQL |
| **Python** | pandas, scikit-learn, SQLAlchemy, matplotlib, seaborn |
| **Machine Learning** | Classification, class imbalance handling, model evaluation, threshold tuning |
| **Data Engineering** | PostgreSQL integration, automated pipelines, cron job scheduling |
| **Data Analysis** | EDA, visualisation, business framing of model trade-offs |

---

## What I'd Do Next

- **Concept drift monitoring** — fraud patterns shift over time; add a rolling retrain trigger when model performance degrades
- **SHAP explainability** — feature contributions per transaction so fraud analysts understand *why* the model flagged something
- **XGBoost / LightGBM** — likely to outperform Random Forest on this tabular dataset
- **Airflow orchestration** — replace cron with a proper DAG for dependency management and alerting
- **FastAPI scoring endpoint** — wrap the trained model for real-time transaction scoring instead of batch runs

---

## Related Projects

- [Olist Data Platform](https://github.com/guiarpi/olist-data-platform) — ELT pipeline + dbt data modelling with DuckDB
- [Customer Churn Prediction](https://github.com/guiarpi/customer-churn-prediction) — similar ML pipeline applied to retention modelling
- [Data Quality AI Framework](https://github.com/guiarpi/data-quality-ai-framework) — anomaly detection with LLM-based reasoning layer

---

*Built as part of a data analytics portfolio. Dataset: [Kaggle Credit Card Fraud Detection](https://www.kaggle.com/datasets/mlg-ulb/creditcardfraud).*
