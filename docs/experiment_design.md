# Experiment design — learning which new members are persuadable

This document specifies the randomised experiment that would be needed to train and validate an uplift model for new-member interventions. The observational data in this repository cannot identify persuadable members; this is the design that would, if deployed.

## Objective
Estimate, per intervention, the causal effect on first renewal for new members, and its heterogeneity by activation segment (§5 of README), so that a targeting policy can be trained on *effect* rather than *risk*.

## Unit and population
- Unit: new member (first paid subscription, no prior history).
- Enrolment: at the prediction point (day **[3]**), all new members not opted out of marketing.
- Exclusions: **[TBD — e.g. members already on a promotional plan]**

## Arms (randomised at enrolment, stratified by activation segment and risk decile)
| Arm | Description | Share |
|---|---|---|
| Control | No intervention | **[25%]** |
| Curation | Personalised starter content / trial title | **[25%]** |
| Reminder | Re-engagement message, no offer | **[25%]** |
| Pause offer | Option to pause instead of cancel, surfaced before renewal | **[TBD]** |
| Discount | **[X]**% off first renewal | **[25%]** |

Stratification ensures every segment has enough members in every arm to estimate heterogeneous effects.

## Primary metric
First renewal (binary), measured at the end of the first subscription period.

## Secondary metrics
Activation within intervention window (H1–H3) · usage in first 30 days · revenue at first renewal (captures discount cost) · cancellation-page visits (pause arm).

## Guardrails
- No arm may reduce renewal below control by more than **[1 pp]** at any interim look (sleeping-dog protection).
- Complaint / unsubscribe rate per arm monitored daily.
- Budget cap per arm.

## Sample size
Minimum detectable effect **[2 pp]** on renewal, baseline renewal rate **[TBD from §9-2]**, power 0.8, alpha 0.05, per stratum. Calculation and resulting N per arm: **TBD** (`notebooks/08_experiment_power.ipynb`).

## Duration
One full subscription cycle plus **[14]** days of observation after the last enrolled member's renewal date.

## Analysis plan (pre-registered)
1. Intention-to-treat difference in renewal, per arm vs control, overall and by stratum.
2. Uplift model (T-learner and causal forest) trained on experiment data; evaluated on held-out experiment data with Qini and the balanced alternative proposed in ICML 2025 uplift-evaluation work.
3. Policy value: expected retained revenue of the learned policy vs "target everyone," "target no one," and the current rule, at fixed budget.

## What the persona simulation in this repo is for
It rehearses this analysis plan on synthetic responses so the pipeline and metrics are ready before real experiment data exist. Its numbers are not evidence of effect.
