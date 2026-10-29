# A-B-testing-product-experimentation
Product Experimentation & A/B Testing Framework

End-to-end experimentation pipeline on **50,000 users** (14-day randomized test of a new
"AI template picker" vs the current homepage): SQL metric layer, automated data validation, frequentist
tests, Bayesian Beta-Binomial with Monte Carlo, power / MDE analysis, segment analysis with multiple-testing
control, and a ship / no-ship recommendation delivered through a Power BI dashboard.

> **Data note:**  not findings about real users.

## Questions the framework answers
1. Is the data trustworthy? (duplicates, missing values, sample-ratio mismatch, randomization balance)
2. Did the treatment change CTR, conversion and session duration, and by how much (with uncertainty)?
3. Was the test big enough? What is the smallest effect it could have detected (MDE)?
4. Does the effect hold across devices / plans / user types, or is an average hiding harm?
5. Should we ship?

## Pre-registered decision rule (fixed before looking at results)
- **Primary metric:** conversion. Ship only if p < 0.05 **and** Bayesian P(treatment > control) >= 0.95.
- **Secondary:** click-through rate.
- **Guardrail:** average session minutes; breached if the relative drop is > 3% and p < 0.05.
- Segment decisions use Benjamini-Hochberg-adjusted p-values; an overall "ship" is downgraded if any device
  segment breaches the guardrail.

## Pipeline
```
simulate_experiment.py -> data/experiment_events.csv (50,150 raw rows incl. injected defects)
sql/01..04 (run on SQLite; portable to PostgreSQL) -> clean_events + metric tables
analyze.py -> validation, z / chi-square / Welch, Bayesian, power, segments, decision
outputs/tables/*.csv -> Power BI (powerbi/POWERBI_GUIDE.md)
```

## How to run
```bash
pip install -r requirements.txt
python src/simulate_experiment.py
python src/analyze.py        # writes outputs/tables, outputs/figures; full log in outputs/run_log.txt
```

## Results (from the run included here)
**Data validation:** 150 duplicate exposures removed; 120 missing and 25 invalid session durations excluded from the
session metric; SRM check passed (25,250 vs 24,750, p = 0.025 > 0.001 threshold); arms balanced on device, plan
and user type (p = 0.10 / 0.40 / 0.77); SQL and Python conversion counts reconcile.

**Overall (25,250 control vs 24,750 treatment):**

| Metric | Control | Treatment | Relative lift (95% CI) | p-value |
|---|---|---|---|---|
| Conversion | 3.94% | 4.51% | +14.3% (+5.1% to +24.3%) | 0.0017 |
| CTR | 11.77% | 13.56% | +15.2% (+10.0% to +20.6%) | < 0.0001 |
| Avg session (min) | 11.04 | 10.74 | -2.8% (-4.4% to -1.1%) | 0.0010 |

Bayesian (Beta(1,1) prior, 200K Monte Carlo draws): P(treatment > control) = 99.9% for conversion and ~100%
for CTR; 95% credible interval for conversion lift +5.2% to +24.4%. Conclusions are unchanged under a Jeffreys
prior and an informative prior (`prior_sensitivity.csv`). The z-test and chi-square test are mathematically
equivalent for two proportions (same p-value); both are reported.

**Power:** at 25,250 users per arm the test could detect a conversion change of about 0.50 percentage points
(about 13% relative) with 80% power; for CTR about 0.82 pp. Detecting a 5% relative conversion lift would need
about 157K users per arm.

**Segments (conversion):**

| Segment | Control | Treatment | Relative lift | BH-adj. p |
|---|---|---|---|---|
| Desktop | 3.88% | 5.06% | +30.4% | < 0.0001 |
| Mobile | 3.82% | 3.86% | +1.0% | 0.89 |
| Tablet (n = 1,691 control) | 5.09% | 3.63% | -28.7% | 0.055 |

Desktop and mobile conversion effects differ significantly (p = 0.002), and mobile session time drops 8.6%
(p < 0.0001), breaching the guardrail. The tablet estimate is a noisy small-sample result, not a finding:
each arm has only about 60-90 conversions, so the correct call is "inconclusive".





## Layout
```
src/      simulate_experiment.py, analyze.py
sql/      01_clean_events ... 04_daily_metrics, postgres_setup.sql
data/     experiment_events.csv (generated)
outputs/  tables/ (Power BI inputs), figures/ (PNG), run_log.txt
powerbi/  POWERBI_GUIDE.md
```
