# Fraud Detection — Findings

> Analysis of 284,807 card transactions (492 confirmed frauds, 0.173%).
> All figures from executed notebooks; every assumption is stated rather than implied.

---

## Summary for a decision-maker

A naive rule-based fraud system was **14× more expensive than switching it off entirely**.
Rebuilding it around expected cost rather than intuition cut the review queue from 100%
of transactions to 1.3%, and a tuned model reduced total cost by a further third.

| Approach | Transactions flagged | Fraud caught | Cost per 57k transactions |
|---|---|---|---|
| Do nothing | 0 | 0% | €10,645 |
| Original three rules | 100% | 100% | **€170,886** |
| Tuned rule (anomaly only) | 1.3% | 89% | €3,532 |
| Model at default threshold | 0.14% | 76% | €4,719 |
| **Model at cost-optimal threshold** | **0.19%** | **89%** | **€2,151** |

The recommended configuration catches 89% of fraud while sending fewer than 2 in 1,000
transactions for review.

---

## 1. The original rule system cost more than it saved

Three rules were built on plausible intuitions: large transactions are suspicious,
rapid-fire transactions are suspicious, statistically abnormal transactions are
suspicious. Measured against a cost model, two of the three destroyed value.

| Rule | Flagged | Precision | Net value vs doing nothing |
|---|---|---|---|
| Anomalous features | 1,087 | 33.0% | **+€33,276** |
| High value (95th pct) | 14,232 | 0.30% | −€7,905 |
| Velocity (60s window) | 284,807 | 0.17% | **−€794,293** |

The velocity rule flagged **every transaction in the dataset**. Its precision equalled
the base fraud rate exactly — it carried no information whatsoever.

**Why it fails is structural, not a matter of tuning.** A velocity check asks whether a
*specific customer* is transacting unusually often. This dataset is anonymised and has no
customer identifier, so there is no entity to group by, and the rule collapses into
"were there any two transactions anywhere within 60 seconds?" At ~1.6 transactions per
second the answer is always yes. Windows from 1 second to 300 seconds were tested; none
recovered it.

**Retired, with the reinstatement condition documented in the stored procedure.**

---

## 2. The typical fraud is tiny, which inverts a common assumption

| | Value |
|---|---|
| Mean fraud | €108.62 |
| **Median fraud** | **€10.34** |
| Largest fraud | €1,809.68 |
| Share of fraud value in the 10 largest cases | 59.6% |

The mean sits ten times above the median: a handful of large frauds, but the *typical*
fraud is very small. The highest-confidence alerts are dominated by transactions of
€0.01 to €1.63 — a pattern consistent with **card testing**, where stolen card numbers
are validated with micro-transactions before anything large is attempted.

This explains the high-value rule's failure directly. It encoded "large transaction =
suspicious." In this data something closer to the opposite holds.

**Practical consequence:** cost models must weight each missed fraud by its *actual*
value. Using the average would treat a €0.01 test transaction and a €1,809 loss as
equivalent.

---

## 3. Rule selection must be judged on marginal contribution

After re-tuning, the high-value rule became profitable on its own — €52,997 against
€60,128 for doing nothing, a €7,131 saving. It was still excluded.

| Configuration | Cost |
|---|---|
| Anomaly rule alone | **€23,494** |
| Anomaly + high-value | €34,345 |

Adding a profitable rule made the ensemble **€10,851 worse**. The anomaly rule had
already caught the frauds high-value would find, so its marginal contribution was 5,467
additional reviews for 5 additional frauds.

**Standalone value and marginal value are different questions.** Only the second should
drive selection. Rules are now chosen by greedy forward selection on total cost.

---

## 4. Optimising for cost, not F1

F1 weights precision and recall equally. In fraud they are not equal: a missed fraud
costs the transaction value, a false alarm costs a few minutes of review — a **36:1
asymmetry** on this data.

| Threshold | Basis | Fraud caught | Cost |
|---|---|---|---|
| 0.50 | Framework default | 74 / 98 | €4,719 |
| 0.255 | Maximises F1 | 82 / 98 | €2,287 |
| **0.125** | **Minimises cost** | **87 / 98** | **€2,151** |

Cost-optimisation saves **€2,568 per 57,000 transactions** against the default —
roughly **€12,800** across the full dataset.

*Assumption:* analyst review costs €3.00 (~2 minutes of loaded labour). This is the only
invented figure in the analysis. Sensitivity testing across €1–€20 shows the optimal
threshold moves, but the cost penalty for holding a fixed threshold stays small — the
cost curve is flat near its minimum.

---

## 5. Review capacity is the more useful frame

Fraud teams have headcount, not thresholds. Ranking every transaction by fraud
probability and taking the top *k*:

| Reviews per period | Fraud caught | Precision | Fraud value recovered |
|---|---|---|---|
| 25 | 24.5% | 96% | 18% |
| 50 | 50.0% | 98% | 34% |
| **100** | **84.7%** | **83%** | **81%** |
| 300 | 90.8% | 30% | 89% |
| 1,000 | 90.8% | 9% | 89% |

**80% of all fraud is reachable by reviewing 84 transactions — 0.15% of traffic.**

Two things follow. Returns diminish sharply past ~100 reviews: going from 100 to 1,000
adds six percentage points of detection at a tenth of the precision. And there is a hard
ceiling — **9 of 98 frauds are never caught at any review budget**. They are invisible to
this model, and no amount of capacity fixes that.

---

## 6. The model generalises across time; the threshold does not

Metrics from a random train/test split are optimistic, because shuffling lets the model
train on later transactions and test on earlier ones. Re-evaluating in strict time order:

| Metric | Random split | Temporal split |
|---|---|---|
| Precision | 0.972 | **0.987** |
| Recall | 0.696 | 0.676 |
| F1 | 0.811 | 0.802 |

**Model performance barely moved** — leakage was not material here.

The threshold is a different story. Tuned on the earlier period it selected **0.375**;
the optimal value for the later period was **0.055**.

| Strategy | Cost on the future period |
|---|---|
| No tuning (0.50) | €4,919 |
| Tuned on past, deployed | €4,585 |
| Hindsight-optimal | €2,573 |

Threshold tuning captured €334 of an available €2,347 — leaving the deployed system
**78% above what perfect foresight would have achieved**. The fraud rate fell from 0.193%
to 0.126% between periods, moving the cost minimum out from under a fixed threshold.

**Recommendation: recalibrate the threshold on a schedule.** A stale threshold costs more
here than a stale model.

---

## 7. The model's reasoning agrees with the hand-built rule

The anomaly rule uses V14, V4 and V12, chosen by visual inspection during exploratory
analysis. SHAP attribution — derived from the model, which was never told about that
choice — ranks them:

| Feature | SHAP rank (of 30) |
|---|---|
| V14 | **1** |
| V12 | **2** |
| V4 | **3** |

Two independent methods converged on the same three features. That is meaningfully
stronger evidence than either alone.

The next-ranked features the rule does *not* use — V11, V3, V10, V17, V16 — are the
natural candidates for extending it.

---

## Limitations

Stated plainly, because they bound how far these conclusions travel:

- **Review cost is assumed**, not measured. Sensitivity-tested, but still an input.
- **Second-order costs are ignored** — customer churn from a false decline, chargeback
  fees, regulatory exposure, reputational damage. All would raise the cost of a false
  positive and push the optimal threshold up.
- **Recovery is assumed perfect.** A flagged fraud is treated as fully prevented; in
  practice recovery is partial.
- **No customer identifier**, so behavioural and velocity features are impossible.
- **Features are PCA-anonymised.** SHAP identifies which latent signal fired and how
  strongly, never what it represents.
- **48 hours of data.** Enough to show threshold drift, not enough for a real drift study.
- **Rule cutoffs are tuned on the full dataset** and so are mildly optimistic; tuning on
  the training fold only would be stricter.
- **A single train/test cut** is one sample. Walk-forward validation would give a
  confidence interval instead of a point estimate.

---

## Recommended next steps

1. **Recalibrate the threshold on a schedule** — the largest measured loss in the system
2. **Test V11, V3, V10 as a fourth rule component** — SHAP-identified, currently unused
3. **Measure the real review cost** to replace the single assumed input
4. **Add walk-forward validation** for confidence intervals rather than point estimates
5. **Obtain a customer identifier** — it would unlock behavioural features and make
   velocity checks meaningful for the first time
