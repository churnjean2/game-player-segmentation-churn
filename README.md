# New-Member Activation & Early Churn — Subscription Lifecycle Analytics

Predicting which newly subscribed members will fail to activate, comparing how early that prediction can be made, and deciding which intervention moves each at-risk member toward activation. Built end-to-end on ~400M rows of subscription usage data (PySpark, LightGBM, MLflow), with a feature abstraction layer designed to port to gaming subscription data (Steam).

> Status: in progress. Sections marked **TBD** are filled in as results land. Decision log in [`docs/decisions.md`](docs/decisions.md).

---

## 1. Business problem

Newly subscribed members are the most expensive to acquire and the most likely to leave. Many pay once, barely use the service, and never renew. The onboarding team can act during the first subscription period, but does not know which new members are at risk or what would actually make them stay.

**The question this project answers:**
*Which new members will fail to activate, how early can we know, and what intervention moves them toward activation before their first renewal?*

## 2. Decision this supports

A risk score for every new member, produced at a fixed point after first payment, plus a recommended intervention type for the onboarding team. The output is a targeting list, not a dashboard.

## 3. Why new members, not all members

Existing members have habits and history; the question there is "what changed." New members have neither; the question is "did they ever find value." Different signals, different interventions, different economics — acquisition cost is already sunk, so every early churner is a full loss on that spend.

## 4. Definitions

| Term | Definition | Rationale |
|---|---|---|
| New member | First paid subscription within the observation window, no prior subscription history | Isolates the onboarding problem from re-activation |
| Early churn | No renewal at the end of the first subscription period | Directly observable in transactions; no proxy needed |
| Activation | Early behaviour most predictive of renewal — see hypotheses below | Defined from data, not assumed |
| Prediction point | Day 0, day 3 and day 7 after first payment, compared | Earlier = more time to intervene, less signal. The trade-off is a result of this project, not an input |
| Intervention window | Day of prediction until **[N]** days before first renewal date | Team needs lead time to act |

## 5. Activation hypotheses (to be tested)

The working assumption: a new member who finds nothing they like on day 0 does not come back. Three measurable forms of that, with a domain-neutral layer so the same pipeline runs on gaming data.

| ID | Abstract signal | KKBox measurement | Steam mapping |
|---|---|---|---|
| H1 | Content found | Completed-play ratio (`num_100` / total plays) on day 0 above threshold; high `num_unq` with low completion = still searching | ≥ 2h in first title (refund threshold) |
| H2 | Return | Any session on day 1 | Same |
| H3 | Depth | Cumulative `total_secs` in first 3 days above threshold | First-week playtime |

Thresholds are **TBD** — set where renewal rate separates most sharply, reported with the analysis.

## 6. Intervention hypotheses

Interventions are matched to *why* activation failed, not applied uniformly. Which segment receives which is a model output, not an assumption.

| Failure mode | Intervention | Cost assumption |
|---|---|---|
| Content not found (H1 fails) | Curation / recommended starter content; free trial title in gaming | **[TBD]** |
| No return (H2 fails) | Reminder / re-engagement message | ≈ 0 |
| Activated but does not renew | Discount | **[X]**% of subscription fee |

The economic claim being tested: a discount given to a member who never found content is wasted spend. If H1 holds, the targeting rule should route those members to curation instead.

## 7. Baseline to beat

**[Assumed current practice: e.g. discount sent to all new members with no activity in the last 7 days.]**
Success is not AUC. Success = higher retained revenue per unit of intervention spend than this rule, at the same budget.

## 8. Data

KKBox subscription data (Kaggle, WSDM 2018): member profiles, ~22M transactions (renewals, cancellations, payment method, auto-renew flag), ~400M daily usage logs.

Chosen because its lifecycle structure — subscribe → use → renew or lapse, with daily activity logs — is the same as gaming subscription services such as PlayStation Plus. Music and games differ in content; the activation layer in §5 is the abstraction that carries across.

Development on a fixed-seed 5% member sample; final run on full data.

## 9. Approach

1. **Cohort construction** — identify new members, attach first payment, first renewal date, renewal outcome.
2. **Activation analysis** — test H1–H3 against renewal rate; fix thresholds; report which hold.
3. **Feature pipeline (PySpark)** — point-in-time features at day 0 / 3 / 7. Leakage check: no feature may use data after the prediction point.
4. **Models** — logistic regression baseline → LightGBM (tuned, calibrated, SHAP) → sequence model on daily logs where history allows. Reported per prediction point.
5. **Lead-time vs precision** — one chart: how much predictive power is lost by predicting earlier, and where the team should sit.
6. **Member value** — expected first-year revenue per new member (BG/NBD + Gamma-Gamma, compared with a direct regression).
7. **Targeting & budget** — rank by risk × value × expected intervention effect − cost; simulate retained revenue against the baseline rule under a fixed budget.
8. **Counterfactual check** — persona-based simulation to stress-test intervention assumptions where observational data has no counterfactual. Clearly labelled as simulation; used for validation, not for training.
9. **MLOps** — MLflow tracking and registry, batch scoring script, containerised scoring endpoint, drift check (PSI).
10. **Steam port** — rerun steps 3–5 on Steam session data through the abstraction layer to demonstrate portability.

## 10. Results

**TBD.** Will report: activation hypotheses confirmed/rejected · model performance per prediction point (PR-AUC, calibration, top-decile lift) · lead-time trade-off · simulated retained revenue vs baseline · what did *not* work and why.

## 11. Out of scope

Personalised recommendation engines, real-time scoring, members beyond their first subscription period.

## 12. Repository layout

```
├── README.md
├── docs/
│   ├── decisions.md        # why each choice was made, including rejected ones
│   ├── executive_summary.md # one page, non-technical
│   └── technical_report.md
├── notebooks/              # numbered by step in §9
├── src/                    # reusable pipeline code
│   ├── cohort.py
│   ├── activation.py       # abstraction layer (H1–H3 definitions per domain)
│   ├── features.py
│   ├── models.py
│   └── targeting.py
├── steam_port/
└── mlops/
```

## Author

Chunjin Kim — MSc Big Data, Sungkyunkwan University · [LinkedIn](https://linkedin.com/in/chunjinkim) · [GitHub](https://github.com/churnjean2)
