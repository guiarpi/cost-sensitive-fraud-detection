# Fraud Detection — Expansion Plan

> **Scope:** one focused week · analytical depth
> **Thesis:** move the project from *"I built a fraud detector"* to
> *"I can tell you what it's worth in euros, and at what review capacity."*

---

## Why this direction

The project's distinguishing quality is already measurement discipline — every rule
benchmarked, the ROC-AUC inversion surfaced, the self-join diagnosed rather than
tolerated. The expansion should sharpen that edge rather than widen the surface area.

Three genuine analytical gaps remain, and each is the kind of thing most junior
portfolios never touch:

| Gap | Why it matters | Currently |
|---|---|---|
| No cost model | F1 treats a missed fraud and a false alarm as equally bad. In reality the asymmetry is **41:1**. | Threshold optimises F1 |
| Random train/test split | Fraud is temporal. A random split lets the model see hour 40 while predicting hour 10 — lookahead leakage. | `train_test_split(..., stratify=y)` |
| Rules flag 100% of data | All three rules combined flag 284,807 of 284,807 transactions. A system that flags everything flags nothing. | Unresolved, undocumented |

Fixing these produces genuinely new findings rather than more charts of the same
finding — which is the difference between depth and padding.

**Deliberately out of scope this week:** XGBoost (Random Forest already at 96%
precision; marginal gain, real time cost), Airflow/productionisation (separate project,
would read as CV padding bolted onto notebooks), dashboards (worth doing, but after the
analysis it would visualise).

---

## Step 0 — Commit the current state first

**Do this before anything else.** What's on GitHub right now is actively wrong: it
publishes model metrics no run produced, and contains two bugs that make notebooks
unrunnable. Expansion takes a week; leaving that public meanwhile is the larger risk.

```bash
cd "~/Documents/Claude/Github Projects/Fraud-Detection"

rmdir olist-product-analytics 2>/dev/null   # stray empty directory
git add -A
git commit -m "fix: sargable range joins, PL/pgSQL FORMAT specifiers, ml_predictions join key

- Rewrite ABS() self-joins as indexed BETWEEN ranges (2h09m unfinished -> 2.6s)
- Add indexes on transactions(Time) and (Time, Amount)
- Fix FORMAT('%.4f') -> '%s'; Postgres FORMAT only supports %s/%I/%L/%%
- Carry transaction_id through the train/test split so ml_predictions can join back
- Load DB credentials from .env in all four notebooks
- Replace README's placeholder metrics with real results from a full run"

git push origin main
git checkout -b expansion/cost-model      # expansion work happens here
```

Keep `main` shippable at all times. Merge each day's work only when the notebook runs
clean end to end.

---

## Day 1–2 · Cost model and alert budget

**New file:** `04-cost-optimisation.ipynb`

This is the highest-value addition in the plan. It reframes the entire project from a
classification exercise into a business decision.

### The grounding numbers (already derivable from your run)

| Quantity | Value | Source |
|---|---|---|
| Total fraud value | €60,127.97 | `fraud_reports.fraud_amount` |
| Fraud count | 492 | `fraud_reports.total_fraud` |
| **Mean fraud value** | **€122.21** | derived |
| Analyst review cost | €3.00 (assumed, ~2 min loaded labour) | stated assumption |
| **Cost asymmetry** | **41 : 1** | derived |

### Tasks

**1.1 — Build the cost matrix.** Define the cost of each confusion-matrix cell.
False negative = mean fraud value (or better, the *actual* `Amount` of that specific
transaction — you have it, so use it rather than the mean). False positive = analyst
review cost. True negative = 0. True positive = review cost, since it still gets
reviewed.

**1.2 — Plot expected cost against threshold.** Sweep 0.01→0.99 and compute total
expected cost at each point. The minimum of that curve is the operating threshold.
Here is what three candidate points already look like on your holdout:

| Threshold | Recall | Precision | Missed | False alarms | Fraud loss | Review cost | **Total** |
|---|---|---|---|---|---|---|---|
| 0.50 (default) | 0.755 | 0.961 | 24 | 3 | €2,933 | €231 | **€3,164** |
| 0.26 (best F1) | 0.837 | 0.932 | 16 | 6 | €1,956 | €264 | **€2,220** |
| ~0.02 (LR-like) | 0.918 | 0.061 | 8 | 1,388 | €977 | €4,434 | **€5,411** |

There is a real optimum between these. Finding it is the deliverable — and note the
F1-optimal threshold already beats the default by ~€944 on a 56,962-row holdout, which
annualises into a number worth saying out loud.

**1.3 — Use per-transaction amounts, not the mean.** Weighting each false negative by
its actual value is more correct and will shift the optimum — high-value frauds should
pull the threshold down. This distinction is itself an interview talking point.

**1.4 — Sensitivity analysis.** The €3 review cost is assumed. Show how the optimal
threshold moves as that assumption varies from €1 to €20. Demonstrating that you know
which of your inputs is soft is a maturity signal.

**1.5 — Precision@k / alert budget.** Real fraud teams have fixed review capacity, not
a threshold. Reframe: *"if the team can review 200 transactions a day, which 200, and
what share of fraud do we catch?"* Plot fraud caught against review budget. This is the
most operationally literate framing available and almost nobody does it.

**Success criterion:** you can state the cost-optimal threshold, what it saves against
the default, and how that answer changes if the review-cost assumption is wrong.

---

## Day 3 · Temporal validation

**New file:** `05-temporal-validation.ipynb`

### The problem

```python
train_test_split(X, y, test_size=0.2, random_state=42, stratify=y)
```

This shuffles. The dataset spans 48 hours via the `Time` column, so the model trains on
transactions from hour 40 and is evaluated on hour 10 — it sees the future. Every metric
you currently report is therefore mildly optimistic.

### Tasks

**3.1 — Split by time, not at random.** Train on the first 70% by `Time`, test on the
last 30%. No shuffling, no stratification.

**3.2 — Retrain and compare honestly.** Report both evaluations side by side. Expect
performance to drop — and *say so*. A candidate who reports that their honest number is
worse than their optimistic one, and explains why, is more credible than one with a
higher number and no methodology.

**3.3 — Check whether fraud rate is stable across time.** Your `fraud_features` table
already shows fraud rate varying from 0.13% to 0.60% across six-hour buckets — roughly
a 4.7× swing. That's the beginning of a drift story: if the base rate moves that much
in 48 hours, a threshold tuned on one period may be wrong for the next.

**3.4 — Write the caveat into the README.** State which numbers come from the random
split and which from the temporal split.

**Success criterion:** two evaluation regimes reported side by side, with an explanation
of why the honest one is lower.

---

## Day 4 · Fix the rules layer

**Modify:** `01-stored-procedure.ipynb`

### The problem

`flagged_transactions` contains 284,807 rows against a `transactions` table of 284,807
rows. The three rules collectively flag **every single transaction**. This is currently
in the repo undocumented and unresolved — the single weakest thing about the project.

### Tasks

**4.1 — Retire or repair the velocity rule.** It flags 269,698 transactions at 0.04%
precision, which is worse than random. Two defensible options:

- *Retire it* and document why — the dataset is anonymised, there is no customer
  identifier, and a velocity check without an entity to group by is just a clock.
- *Repair it* by constructing a synthetic entity key (for example, clustering on the
  PCA components to approximate a cardholder) and re-measuring. More ambitious, and it
  may still fail — which is a legitimate result.

Recommended: retire, document, and note the repair as future work. The negative result
is already strong interview material; a half-working repair would weaken it.

**4.2 — Re-tune the high-value threshold using the Day 1 cost model.** The 95th
percentile is arbitrary. Sweep the amount threshold, apply the cost function, and pick
the value that minimises cost. It may well conclude the rule isn't worth running at all
at 0.30% precision — also a legitimate finding.

**4.3 — Report rules as an ensemble, not just individually.** Combined precision,
combined recall, and combined cost. Then compare that ensemble against the model on
identical terms.

**4.4 — Express every rule in euros.** Convert the precision/recall table into a cost
table. "The anomaly rule saves €X per 10,000 transactions; the velocity rule costs €Y"
is a far stronger sentence than a precision percentage.

**Success criterion:** `flagged_transactions` contains a defensible subset, and every
rule has a euro value attached.

---

## Day 5 · Explainability and consolidation

### Tasks

**5.1 — SHAP on the Random Forest.** A summary plot for global feature importance, plus
two or three force plots for individual flagged transactions. The operational argument
matters more than the visual: an analyst who can't interrogate a flag learns to ignore
it, so per-prediction attribution is what makes the model's output as reviewable as a
rule.

**5.2 — Sanity-check SHAP against the rules.** Your anomaly rule uses V14, V4 and V12,
chosen from EDA. If SHAP independently ranks those features highly, that's convergent
validation of your feature selection — worth stating explicitly.

**5.3 — Write `FINDINGS.md`.** One page, stakeholder-facing, no code. The cost-optimal
threshold, what it saves, the honest temporal number, the rules verdict, and the three
limitations you'd flag before anyone acted on this.

**5.4 — Update `README.md`.** New results, plus the cost framing in the opening.

**5.5 — Refresh `FraudDetection_InterviewPrep.md`.** New numbers, and a new section on
cost-sensitive evaluation — likely your strongest material once it exists.

**5.6 — Merge and push.**

```bash
git checkout main && git merge expansion/cost-model && git push origin main
```

---

## What you'll be able to say afterwards

Before: *"I built a fraud detection system with rules and machine learning."*

After: *"I found the threshold that minimises expected cost rather than maximising F1,
because a missed fraud costs 41 times what a false alarm does. Against the default 0.5
cutoff that's roughly €940 saved per 57,000 transactions. I also re-validated the whole
thing on a time-ordered split, because the random split was leaking future information —
the honest numbers are lower than the ones I'd have reported otherwise."*

The second answer is a different candidate.

---

## Running list of assumptions to state, not hide

Interviewers respect stated assumptions and punish hidden ones. Keep these visible in
both the notebook and the README:

- €3 analyst review cost is assumed, not measured — hence the sensitivity analysis
- Mean fraud value is computed from this dataset only and won't generalise
- No customer identifier exists, so no behavioural or velocity features are possible
- Features are PCA-anonymised, so SHAP shows *which* component matters, never *why*
- 48 hours of data means drift analysis is directional at best
- The cost model ignores second-order effects — customer churn from a false decline,
  chargeback fees, regulatory exposure
