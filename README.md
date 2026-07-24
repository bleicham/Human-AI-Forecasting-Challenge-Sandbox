# SISMID Human-AI Influenza Forecasting Mini-Challenge

A sandbox forecasting challenge for the SISMID *Human-AI Teaming Approach to Infectious
Disease Modeling* lab. It is modeled on the US CDC's
[FluSight challenge](https://github.com/cdcepi/FluSight-forecast-hub) and built to
[hubverse](https://hubverse.io/) standards, so that participants can work through a real
collaborative-forecasting pipeline end to end: from raw surveillance data, to a submitted
probabilistic forecast, to an evaluated, directly comparable score.

The hub structure was adapted from code by Nick Reich
([original](https://github.com/reichlab/sismid-ili-forecasting-sandbox)), with the target
changed from outpatient ILI to laboratory-confirmed influenza hospital admissions.

> [!IMPORTANT]
> **This is an educational sandbox, not operational FluSight.** It deliberately mirrors
> the structure of a real forecasting challenge for teaching purposes, but it is **not**
> the CDC FluSight challenge, every forecast in it is **retrospective**, and nothing here
> is intended to inform public health decisions.

## The challenge

Teams forecast the near-term burden of influenza in the United States and submit their
predictions in a single standard format, so that different modeling approaches can be
compared on the same target, horizons, and data.

- **Prediction target:** `wk inc flu hosp` — the number of new laboratory-confirmed
  influenza hospital admissions reported to the US CDC's
  [National Healthcare Safety Network (NHSN)](https://www.cdc.gov/nhsn/index.html) for an
  epidemiological week (EW), at the national level (`location = "US"`).
- **Horizons:** 1, 2, and 3 weeks ahead of each reference date. Submitting all three is
  encouraged but not required.
- **Reference dates:** the 2025-2026 influenza season, `2025-09-27` through `2026-05-23`.
- **Output:** 23 quantiles per target, following the hubverse / FluSight quantile format.

We use the [CDC MMWR epidemiological week](https://wwwn.cdc.gov/nndss/document/MMWR_Week_overview.pdf)
definition, which runs Sunday through Saturday. The target end date for a prediction is
the Saturday that ends the EW being predicted:

> **target end date = reference date + horizon × 7 days**

Standard packages convert between calendar dates and epidemic weeks, for example
[MMWRweek](https://cran.r-project.org/web/packages/MMWRweek/) and
[lubridate](https://lubridate.tidyverse.org/reference/week.html) for R, and
[pymmwr](https://pypi.org/project/pymmwr/) and [epiweeks](https://pypi.org/project/epiweeks/)
for Python.

> [!NOTE]
> **On horizons.** This hub uses **horizons 1 through 3**, with the reference date being
> the Saturday ending the most recent week of data used by the forecast. Operational
> FluSight uses horizons −1 through 3 relative to a Wednesday submission deadline, so its
> horizon 0 is a nowcast of a week that has not yet been reported. Forecasts produced here
> are therefore **not** directly comparable to FluSight submissions without re-indexing.
> This is a deliberate simplification for teaching.

## Submitting a forecast

Submissions follow the hubverse model-output schema:

```
model_id, reference_date, target, horizon, location, target_end_date, output_type, output_type_id, value
```

Each team places its forecasts in `model-output/<team>-<model>/`, one CSV per reference
date named `<reference_date>-<team>-<model>.csv`, with a matching
`model-metadata/<team>-<model>.yml`. (Note the `<team>-<model>` model id requires the
hyphen.)

## Evaluation

Forecasts are scored with [`scoringutils`](https://epiforecasts.io/scoringutils/) against
`target-data/oracle-output.csv`. Reported metrics are:

- **Weighted Interval Score (WIS)** — overall skill across the full predictive distribution
- **Absolute error of the median** — point accuracy
- **95% prediction-interval coverage** — calibration

Because every team forecasts the same target in the same format, these scores are directly
comparable. Two things to know before reading them:

- `scoringutils::score()` returns a per-forecast `ae_median`. **MAE only exists after
  averaging** across forecasts, so it appears in a by-horizon or by-model summary, not in
  the raw score output.
- `scoringutils` does **not** compute 95% interval coverage by default for the 0.025 /
  0.975 quantiles. It must be requested explicitly.

## Target data

Two files in `target-data/` hold the observed data:

- **[`target-data/time-series.csv`](target-data/time-series.csv)** — the observed weekly
  admissions series, shown alongside forecasts by the
  [predtimechart](https://docs.hubverse.io/en/latest/user-guide/dashboards.html) dashboard.
- **[`target-data/oracle-output.csv`](target-data/oracle-output.csv)** — the scoring truth
  each forecast is evaluated against.

> [!NOTE]
> **The current sandbox data is finalized, not vintaged.** The committed target files use
> settled (season-final) admission values, with no `as_of` versioning. This keeps the
> teaching setup simple and the scores reproducible, at the cost of the real-time backfill
> effect described below. Wiring [`scripts/get_target_data.R`](scripts/get_target_data.R)
> to pull *versioned* NHSN data — so the observed line matches what each forecaster
> actually saw at their reference date — is the intended next step.

### Why timing matters

Because this hub is retrospective, forecasts are generated **after** the season, using
data we can look up today. NHSN admissions are **revised after they are first published**
(this is called **backfill**), so "the admissions count for week *W*" is not a single
number — it depends on *when you ask*. A model that is not careful to use only the data
that would actually have been available at its `reference_date` will look more accurate
than any real-time forecaster could, because it can "see" values that were still settling
when the forecast date arrived. The intended honesty mechanism is a data-availability
cutoff:

> **data cutoff = reference_date + 9 days**

<details>
<summary><b>Full reporting-calendar and regeneration detail (for hub maintainers)</b></summary>

Read this section before changing
[`scripts/get_target_data.R`](scripts/get_target_data.R).

**The NHSN reporting calendar.** Three facts drive everything below:

- Facilities report a Sunday-through-Saturday week to NHSN by **Tuesday 11:59pm PT** of
  the following week.
- CDC publishes **preliminary** figures on **Wednesday** (about 4 days after the week
  ends) and **finalized** figures on **Friday or Saturday** (about 6 to 7 days after the
  week ends). Each release adds the week just ended.
- Revisions to a given week continue after first publication.

**The real-time cutoff.** Each forecast is labelled by its `reference_date`, the Saturday
ending the most recent epiweek of data used. The cutoff rule
(`data cutoff = reference_date + 9 days`) works because the reference week is first
published on the Friday or Saturday after it ends (`reference_date + 6` or `+ 7`), so by
the Monday cutoff (`reference_date + 9`) the reference week is available. The *next* week's
first preliminary release does not appear until Wednesday (`reference_date + 11`), so it is
correctly excluded. A forecast "as of" `reference_date` is therefore built from data
**through the reference week, at its first-reported (unrevised) values**.

**Worked example** (national, `reference_date = 2026-01-03`):

| week ending | value the forecaster had (as of the cutoff) | season-final value (scoring truth) |
|---|---|---|
| `2025-12-27` | `<TBD>` | `<TBD>` |
| `2026-01-03` | `<TBD>` (first reported `2026-01-09`) | `<TBD>` |

> To be filled in from an actual versioned pull. Pick a reference date near the season
> peak, where backfill is largest and the teaching point lands hardest.

</details>

## Acknowledgments

This repository follows the guidelines and standards outlined by
[the hubverse](https://hubverse.io), which provides a set of data formats and open-source
tools for modeling hubs. The hub is adapted from Nick Reich's
[SISMID ILI forecasting sandbox](https://github.com/reichlab/sismid-ili-forecasting-sandbox).
