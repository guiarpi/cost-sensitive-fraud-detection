# Fraud Detection System — PostgreSQL + Python + Machine Learning

An end-to-end fraud detection pipeline built on 284,807 real transactions — rule-based
detection, machine learning, cost-sensitive optimisation and explainability, with every
decision priced in euros rather than argued in F1 points.

**[→ Read the findings](FINDINGS.md)**

---

## The Problem

Fraud detection is a cost-optimisation problem disguised as a classification problem.
Catching more fraud is trivial — flag everything. The difficulty is catching fraud
without burying an operations team in false positives.

Most treatments stop at "the model scores 0.98 ROC-AUC." This project asks the questions
that come after: *what does a missed fraud actually cost, what does a false alarm cost,
what threshold follows from that, and does any of it survive contact with time?*

| Stage | Approach | What it adds |
|---|---|---|
| 1 | Rule-based detection | Fast, explainable, auditable — and measurable |
| 2 | Machine learning | Catches patterns nobody wrote down |
| 3 | Automated reporting | Trends and alerts without manual querying |
| 4 | Cost optimisation | Replaces F1 with expected cost in euros |
| 5 | Temporal validation | Removes lookahead leakage; exposes threshold drift |
| 6 | Rule optimisation | Prices each rule; retires the ones that destroy value |
| 7 | Explainability | Makes the model as reviewable as the rules |

---

## Headline Results

| Approach | Flagged | Fraud caught | Cost per 57k transactions |
|---|---|---|---|
| Do nothing | 0 | 0% | €10,645 |
| Original three rules | **100%** | 100% | **€170,886** |
| Tuned rule (anomaly only) | 1.3% | 89% | €3,532 |
| Model at default threshold 0.50 | 0.14% | 76% | €4,719 |
| **Model at cost-optimal threshold 0.125** | **0.19%** | **89%** | **€2,151** |

Three findings worth the click:

- **A naive rule system cost 14× more than switching it off.** The velocity rule flagged
  every transaction in the dataset at precision exactly equal to the base rate — zero
  information. It fails structurally: anonymised data has no customer identifier, so
  there is no entity to compute velocity against.
- **The model with the better ROC-AUC was the worse model.** Logistic Regression scored
  0.972 against Random Forest's 0.953, at 6% precision versus 96%. Under 578:1 imbalance
  ROC-AUC flatters over-flagging.
- **The model generalises across time; the threshold does not.** F1 moved 0.811 → 0.802
  under temporal validation, but the cost-optimal threshold moved 0.375 → 0.055 — leaving
  a deployed system 78% above achievable cost.

---

## Dataset

**[Kaggle — Credit Card Fraud Detection](https://www.kaggle.com/datasets/mlg-ulb/creditcardfraud)**

- 284,807 transactions from European cardholders (September 2013)
- 492 fraudulent transactions (**0.172%** — severe class imbalance)
- Features: `Time`, `V1–V28` (PCA-anonymised), `Amount`, `Class`

> The dataset is not included in this repo due to file size. See Setup below — `kagglehub` downloads it automatically on first run.

---

## Key Results

All figures below are from an actual run over the full 284,807-transaction dataset
(56,962-row stratified holdout containing 98 frauds).

### Rule-based layer

| Rule | Flagged | Caught | Precision | Recall | Lift vs 0.173% base |
|---|---|---|---|---|---|
| Anomalous features (`V14 < -5 OR V4 > 5 OR V12 < -5`) | 1,087 | 359 | **33.0%** | **73.0%** | **191×** |
| High value (`Amount > €365`, 95th pct) | 14,232 | 43 | 0.30% | 8.7% | 1.7× |
| Velocity (two transactions within 60s) | 269,698 | 113 | 0.04% | 23.0% | 0.2× |

Three SQL conditions catch 73% of all fraud at 1-in-3 precision. The velocity rule
flags 95% of the dataset at *worse than random* precision — kept in the repo as a
documented negative result, because the dataset is anonymised and has no customer
identifier to compute velocity against.

### Machine learning layer

| Model | Precision (Fraud) | Recall (Fraud) | F1 | ROC-AUC |
|---|---|---|---|---|
| Logistic Regression (balanced) | 0.061 | 0.918 | 0.114 | **0.972** |
| Random Forest (balanced) | **0.961** | 0.745 | **0.839** | 0.953 |
| Random Forest @ tuned threshold 0.26 | 0.932 | **0.837** | — | — |

**The headline finding is the ROC-AUC trap.** Logistic Regression scores the *higher*
ROC-AUC (0.972 vs 0.953) and is decisively the worse model — 6% precision means 15
false alarms for every genuine fraud. ROC-AUC's false-positive-rate axis is diluted by
a negative class 578× larger than the positive one, so it flatters models that
over-flag. The precision–recall curve and F1 (0.84 vs 0.11) rank them correctly.

**Threshold tuning matters.** Moving the cutoff from the default 0.5 to 0.26 bought
+8.2 points of recall for −2.9 points of precision — catching 8 more frauds per 98 at
the cost of a handful of extra reviews. The default 0.5 is an arbitrary inheritance,
not a decision.

**Does ML earn its complexity here?** Yes, decisively: 93% precision at 84% recall
versus the best rule's 33% at 73%. An analyst reviewing the model's queue finds fraud
in 9 cases out of 10; reviewing the rule's queue, 1 in 3.

*Reproduce: run the notebooks in order. Full evaluation in `02-machine-learning.ipynb`.*

---

## Project Structure

```
cost-sensitive-fraud-detection/
├── 00-eda.ipynb                  # Exploratory analysis & visualisations
├── 01-stored-procedure.ipynb     # Rule-based detection via PostgreSQL stored procedures
├── 02-machine-learning.ipynb     # Logistic Regression vs Random Forest, full evaluation
├── 03-automated-reports.ipynb    # Fraud reports, alerts, cron scheduling
├── 04-cost-optimisation.ipynb    # Expected-cost thresholds, alert budget, precision@k
├── 05-temporal-validation.ipynb  # Time-ordered split; deployment simulation
├── 06-rule-optimisation.ipynb    # Rules priced in euros; marginal selection
├── 07-explainability.ipynb       # SHAP attribution; analyst-facing alert queue
├── FINDINGS.md                   # Stakeholder-facing summary, no code
├── EXPANSION-PLAN.md             # Roadmap for the analytical deepening above
├── plots/                        # Generated visualisation outputs
├── requirements.txt
└── .gitignore
```

Run in numerical order — `01` populates the PostgreSQL tables that `03` and `06` read.
Notebooks `04`, `05` and `07` are self-contained and retrain independently.

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
- Alert logic: triggers when fraud rate exceeds configurable threshold
- Cron job setup for scheduled execution
- **Query optimisation:** the original self-join used `ABS(t2.Time - t1.Time) < 30`,
  which is non-sargable — the planner cannot index it, degenerating into ~40.5 billion
  pair comparisons. Rewritten as an indexable `BETWEEN` range: **2h09m unfinished → 2.6s**

### `04-cost-optimisation.ipynb` — Cost-Sensitive Thresholds
- Explicit cost model: each false negative weighted by its *actual* transaction value
- Threshold chosen by minimising expected cost rather than maximising F1
- Sensitivity analysis across a 20× range of the one assumed input
- Precision@k / alert budget — reframes the model as a staffing decision

### `05-temporal-validation.ipynb` — Honest Evaluation
- Time-ordered split replacing the random one, removing lookahead leakage
- Deployment simulation: tune on the past, apply unchanged to the future
- Quantifies the gap against a hindsight-optimal threshold

### `06-rule-optimisation.ipynb` — Pricing the Rules
- Every rule valued in euros against a "do nothing" baseline
- Velocity rule retired with evidence — swept across windows from 1s to 300s
- Cutoffs re-tuned against cost instead of round numbers
- Greedy forward selection on marginal contribution, not standalone value
- Rules vs model head-to-head on identical terms

### `07-explainability.ipynb` — SHAP
- Global and per-prediction feature attribution
- Cross-validates the hand-built rule: SHAP ranks V14, V12, V4 as its **top three**,
  arrived at independently of the EDA that chose them
- Analyst-facing alert queue with plain-language flag reasons

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
git clone https://github.com/guiarpi/cost-sensitive-fraud-detection.git
cd cost-sensitive-fraud-detection

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
| **SQL** | Stored procedures, self-joins, CTEs, indexing, query-plan reasoning (sargability) |
| **Python** | pandas, scikit-learn, SHAP, SQLAlchemy, matplotlib, seaborn |
| **Machine Learning** | Class imbalance, threshold tuning, cost-sensitive evaluation, explainability |
| **Data Engineering** | PostgreSQL integration, automated pipelines, scheduled execution |
| **Data Analysis** | EDA, visualisation, expected-value modelling, sensitivity analysis |
| **Judgement** | Negative results reported; assumptions stated; marginal vs standalone value |

---

## What I'd Do Next

Ordered by expected value, not by novelty:

- **Scheduled threshold recalibration** — the largest measured loss in the system. The
  deployed threshold sat 78% above achievable cost after a single period of drift.
- **Extend the anomaly rule with V11, V3, V10** — SHAP-identified, currently unused
- **Measure the real analyst review cost** to replace the single assumed input
- **Walk-forward validation** — confidence intervals rather than a point estimate from
  one train/test cut
- **Second-order costs** — churn from false declines, chargeback fees, regulatory
  exposure. All raise the cost of a false positive and would move the optimum.
- **XGBoost / LightGBM** — likely marginal at 96% precision, but worth measuring
- **Airflow orchestration** — cron has no dependency management, retries or failure
  visibility

---

## Related Projects

- [Olist Data Platform](https://github.com/guiarpi/olist-data-platform) — ELT pipeline + dbt data modelling with DuckDB
- [Customer Churn Prediction](https://github.com/guiarpi/customer-churn-prediction) — similar ML pipeline applied to retention modelling
- [Data Quality AI Framework](https://github.com/guiarpi/data-quality-ai-framework) — anomaly detection with LLM-based reasoning layer

---

*Built as part of a data analytics portfolio. Dataset: [Kaggle Credit Card Fraud Detection](https://www.kaggle.com/datasets/mlg-ulb/creditcardfraud).*
