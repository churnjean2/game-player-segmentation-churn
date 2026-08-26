# day-zero — New-Member Activation, Early Churn and Causal Targeting

Which new subscribers fail to reach first value in their first days, which of them an intervention can actually move, and which intervention is right for each. Built end-to-end on ~400M rows of subscription usage data (PySpark, LightGBM, MLflow), with an activation abstraction layer designed to port to gaming subscription data (Steam).

> Status: in progress. Sections marked **TBD** are filled in as results land. Every design choice, including rejected ones, is logged in [`docs/decisions.md`](docs/decisions.md). The experiment this project would run if deployed is specified in [`docs/experiment_design.md`](docs/experiment_design.md).

---

## 1. Business problem

The subscription market has shifted from "acquire at all costs" to "activate and retain." For a service like PlayStation Plus, growth now depends on converting a very large active-user base into paying members and keeping them past their first renewal — at a price point that, after the 2026 increases, is higher for new members than for existing ones. New members therefore face a stricter value test than anyone else in the base, and industry evidence puts the decisive window at roughly the first three days.

Predicting who will leave is no longer the frontier. Field evidence shows that the highest-risk members are often not the ones an intervention can save (Ascarza 2018), that targeting on expected effect rather than churn probability materially raises returns (Lemmens & Gupta 2020; Athey et al. 2025), and that discounts are the default tool but rarely the right one (Devriendt et al. 2021).

**The question this project answers:**
*Among new members, who fails to reach first value in the first days, which of them can an intervention actually move, and which intervention — curation, reminder, pause offer, or discount — is right for each?*

## 2. Decision this supports

A daily list for the onboarding team: every new member scored at a fixed point after first payment, with (a) risk of early churn, (b) expected first-year value, (c) estimated responsiveness to each intervention, and (d) the recommended action, including "do nothing." The output is a targeting policy, not a churn dashboard.

## 3. Why new members, not all members

Existing members have habits and history; the question there is "what changed." New members have neither; the question is "did they ever find value." Bolton (1998) shows early-relationship customers are far more sensitive to a single experience, and Datta et al. (2015) find that early usage intensity is the dominant predictor of retention for trial-acquired customers. Different signals, different interventions, different economics — acquisition cost is already sunk, so every early churner is a full loss on that spend.

## 4. Definitions

| Term | Definition | Rationale |
|---|---|---|
| New member | First paid subscription within the observation window, no prior subscription history | Isolates onboarding from re-activation |
| Early churn | No renewal at the end of the first subscription period | Directly observable in transactions |
| Time-to-value (activation) | Earliest behaviour that separates renewers from non-renewers — hypotheses in §5 | Defined from data, not assumed |
| Prediction point | Day 0, day 3 and day 7 after first payment, compared | Earlier = more lead time, less signal; the trade-off is a result, not an input |
| Intervention window | Prediction point until **[N]** days before first renewal date | Team needs lead time to act |
| Persuadable | A member whose renewal outcome changes because of the intervention | The only group worth spending on |

## 5. Activation hypotheses (to be tested)

Working assumption: a new member who finds nothing they like on day 0 does not come back. Three measurable forms, with a domain-neutral layer so the same pipeline runs on gaming data.

| ID | Abstract signal | KKBox measurement | Steam mapping |
|---|---|---|---|
| H1 | Content found | Completed-play ratio (`num_100` / total plays) on day 0 above threshold; high `num_unq` with low completion = still searching | ≥ 2h in first title (refund threshold) |
| H2 | Return | Any session on day 1 | Same |
| H3 | Depth | Cumulative `total_secs` in first 3 days above threshold | First-week playtime |

Thresholds are **TBD** — set where renewal rate separates most sharply, reported with the analysis.

## 6. Intervention hypotheses

Matched to *why* activation failed. Which segment receives which is a model output, not an assumption.

| Failure mode | Intervention | Cost assumption | Evidence available in this data |
|---|---|---|---|
| Content not found (H1 fails) | Curation / recommended starter content; free trial title in gaming | **[TBD]** | None — simulation + experiment design only |
| No return (H2 fails) | Reminder / re-engagement message | ≈ 0 | None — simulation + experiment design only |
| Activated, price-sensitive | Discount | **[X]**% of fee | Observational: discounted vs. full-price new members |
| Activated, low current need | Pause offer instead of cancel | ≈ 0 | None — simulation + experiment design only |

The economic claim being tested: a discount to a member who never found content is wasted spend; a reminder to a member who has already decided is noise or worse. If H1 holds, the policy should route non-activated members to curation, not discount.

## 7. Baseline to beat

**[Assumed current practice: e.g. discount to every new member with no activity in the last 7 days.]**
Success is not AUC. Success = higher retained revenue per unit of intervention spend than this rule, at the same budget.

## 8. Data

KKBox subscription data (Kaggle, WSDM 2018): member profiles, ~22M transactions (renewals, cancellations, list price vs. amount paid, auto-renew flag), ~400M daily usage logs.

Chosen because its lifecycle — subscribe → use → renew or lapse, with daily activity logs — is structurally the same as a gaming subscription. Music and games differ in content; the activation layer in §5 is the abstraction that carries across.

**Known limitation, stated up front:** there is no randomised intervention data. Causal effects of discounts are estimated observationally (§9-7) with explicit bias caveats; effects of the other interventions cannot be estimated from this data and are handled by simulation and a proposed experiment (§9-8, `docs/experiment_design.md`).

Development on a fixed-seed 5% member sample; final run on full data.

## 9. Approach

1. **Cohort construction** — identify new members, attach first payment, first renewal date, renewal outcome.
2. **Activation analysis** — test H1–H3 against renewal rate; fix thresholds; report which hold and the observed time-to-value distribution.
3. **Feature pipeline (PySpark)** — point-in-time features at day 0 / 3 / 7. Leakage check: no feature may use data after the prediction point.
4. **Risk models** — logistic regression baseline → LightGBM (tuned, calibrated, SHAP) → sequence model on daily logs where history allows. Reported per prediction point.
5. **Lead-time vs precision** — one chart: how much predictive power is lost by predicting earlier, and where the team should sit.
6. **Member value** — expected first-year revenue per new member (BG/NBD + Gamma-Gamma, compared with a direct regression).
7. **Observational treatment effect (discount)** — propensity-score matching and a doubly-robust estimator on discounted vs. full-price new members; heterogeneity by activation segment. Reported as bounded, not causal-certain.
8. **Persuadable identification & experiment design** — persona-based simulation to stress-test intervention assumptions where no counterfactual exists (labelled as simulation, used for validation only), plus a full A/B design for learning an uplift model in production: arms, sample size, duration, primary metric, guardrails.
9. **Targeting policy & budget** — rank by risk × value × estimated effect − cost; simulate retained revenue against the baseline rule at a fixed budget; show what share of members the policy targets versus "target everyone" and "target no one."
10. **MLOps** — MLflow tracking and registry, batch scoring script, containerised scoring endpoint, drift check (PSI).
11. **Steam port** — rerun steps 3–5 on Steam session data through the abstraction layer to demonstrate portability.

## 10. Results

**TBD.** Will report: activation hypotheses confirmed/rejected · time-to-value distribution · model performance per prediction point (PR-AUC, calibration, top-decile lift) · lead-time trade-off · discount effect estimate with sensitivity analysis · policy simulation vs baseline · share of members targeted · what did *not* work and why.

## 11. Out of scope

Personalised recommendation engines, real-time scoring, members beyond their first subscription period, tier upsell.

## 12. Repository layout

```
├── README.md
├── docs/
│   ├── decisions.md          # why each choice was made, including rejected ones
│   ├── experiment_design.md  # the A/B test this project would run if deployed
│   ├── executive_summary.md  # one page, non-technical
│   └── technical_report.md
├── notebooks/                # numbered by step in §9
├── src/
│   ├── cohort.py
│   ├── activation.py         # abstraction layer (H1–H3 per domain)
│   ├── features.py
│   ├── models.py
│   ├── causal.py             # observational effect estimation
│   └── targeting.py
├── steam_port/
└── mlops/
```

## References

- Ascarza, E. (2018). Retention futility: Targeting high-risk customers might be ineffective. *Journal of Marketing Research*, 55(1).
- Athey, S., Keleher, N., & Spiess, J. (2025). Machine learning who to nudge: Causal vs predictive targeting in a field experiment. *Journal of Econometrics*.
- Bolton, R. N. (1998). A dynamic model of the duration of the customer's relationship with a continuous service provider. *Marketing Science*, 17(1).
- Datta, H., Foubert, B., & van Heerde, H. J. (2015). The challenge of retaining customers acquired through free trials. *Journal of Marketing Research*, 52(2).
- Devriendt, F., Berrevoets, J., & Verbeke, W. (2021). Why you should stop predicting customer churn and start using uplift models. *Information Sciences*, 548.
- Fader, P. S., Hardie, B. G. S., & Lee, K. L. (2005). "Counting your customers" the easy way: An alternative to the Pareto/NBD model. *Marketing Science*, 24(2).
- Lemmens, A., & Gupta, S. (2020). Managing churn to maximize profits. *Marketing Science*, 39(5).
- Neslin, S. A. et al. (2006). Defection detection: Measuring and understanding the predictive accuracy of customer churn models. *Journal of Marketing Research*, 43(2).
- Verbeke, W. et al. (2012). New insights into churn prediction in the telecommunication sector: A profit driven data mining approach. *European Journal of Operational Research*, 218(1).
- Hadiji, F. et al. (2014). Predicting player churn in the wild. *IEEE CIG*.

## Author

Chunjin Kim — MSc Big Data, Sungkyunkwan University · [LinkedIn](https://linkedin.com/in/chunjinkim) · [GitHub](https://github.com/churnjean2)
