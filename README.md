# SISMID Human-AI Influenza Forecasting Mini-Challenge

A template Sandbox hub of forecasts based on the [FluSight Challenge](https://github.com/cdcepi/FluSight-forecast-hub) run by the CDC. All data and the repository structure have been formatted according to [hubverse](https://hubverse.io/) standards.

The purpose of this hub is to provide a sandbox environment for training, research, or benchmarking. The hub structure was adapted from code provided by Nick Reich ([original code](https://github.com/reichlab/sismid-ili-forecasting-sandbox)), with the target changed from outpatient ILI to laboratory-confirmed influenza hospital admissions.

## Short-term influenza forecasts

Predictions are quantile forecasts of weekly laboratory-confirmed influenza hospital admissions at the national level during the 2025-2026 season. Unlike FluSight and other operational challenges, all forecasts produced for this hub are retrospective. The hub is set up to receive new forecast submissions for educational purposes.

**Dates:** Reference dates span the 2025-2026 influenza season (`2025-10-04` through `2026-05-23`). These are set in `hub-config/tasks.json` and are the authoritative values; the text here follows that file rather than the other way around.

**Prediction target:** `wk inc flu hosp`, the total number of new laboratory-confirmed influenza hospital admissions reported to NHSN for an epidemiological week, at the national level (`location = "US"`).

Modelers submit retrospective quantile forecasts for the epidemiological week (EW) ending on each reference date, up to 3 weeks ahead. Submitting all three horizons is encouraged but not required. We use the EW specification defined by the [CDC](https://wwwn.cdc.gov/nndss/document/MMWR_Week_overview.pdf), which runs Sunday through Saturday. The target end date for a prediction is the Saturday that ends the EW of interest:

> **target end date = reference date + horizon × 7 days**

Submissions follow the hubverse model output schema: `model_id`, `reference_date`, `target`, `horizon`, `location`, `target_end_date`, `output_type`, `output_type_id`, `value`. The quantile levels accepted by this hub are defined in `hub-config/tasks.json`.

Ground truth target data [is downloaded](scripts/get_target_data.R) using [the epidatr R package](https://cmu-delphi.github.io/epidatr/), which serves issue-versioned NHSN data through the [Delphi Epidata API](https://cmu-delphi.github.io/delphi-epidata/api/covidcast-signals/nhsn.html).

There are standard software packages to convert between dates and epidemic weeks, for example [MMWRweek](https://cran.r-project.org/web/packages/MMWRweek/) and [lubridate](https://lubridate.tidyverse.org/reference/week.html) for R, and [pymmwr](https://pypi.org/project/pymmwr/) and [epiweeks](https://pypi.org/project/epiweeks/) for Python.

### A note on horizons

This hub uses **horizons 1 through 3**, with the reference date being the Saturday ending the most recent week of data used by the forecast. This is a deliberate simplification for teaching. Operational FluSight uses horizons -1 through 3 relative to a Wednesday submission deadline, so its horizon 0 is a nowcast of a week that has not yet been reported. Forecasts produced here are therefore **not** directly comparable to FluSight submissions without re-indexing.

## Data versions, reporting backfill, and forecast timing

NHSN hospital admissions are **revised after they are first published**. As facilities correct and complete their submissions, the admissions count for a given week is updated over the following weeks. This is called **backfill**. Because of it, "the admissions count for week *W*" is not a single number: it depends on *when you ask*. Getting this right matters for a retrospective hub, because a forecaster in real time saw only the (often incomplete) data available at the time, not the settled values we can look up today.

This hub handles versioning deliberately, and it is the source of two easily confused date issues. Read this section before changing [`scripts/get_target_data.R`](scripts/get_target_data.R).

### The NHSN reporting calendar

Three facts drive everything below:

- Facilities report a Sunday-through-Saturday week to NHSN by **Tuesday 11:59pm PT** of the following week.
- CDC publishes **preliminary** figures on **Wednesday** (about 4 days after the week ends) and **finalized** figures on **Friday or Saturday** (about 6 to 7 days after the week ends). Each release adds the week just ended.
- Revisions to a given week continue after first publication and generally settle within about **two months**.

### What data a forecast could actually use (the real-time cutoff)

Each forecast is labelled by its **`reference_date`**, the Saturday ending the most recent epiweek of data used in the forecast. Targets are 1 to 3 weeks *ahead* of that week.

The cutoff rule this hub applies is:

> **data cutoff = reference_date + 9 days**

The arithmetic works because the reference week is first published on the Friday or Saturday after it ends (`reference_date + 6` or `+ 7`), so by the Monday cutoff (`reference_date + 9`) the reference week is available. The *next* week's first preliminary release does not appear until Wednesday (`reference_date + 11`), so it is correctly excluded. A forecast "as of" `reference_date` is therefore built from data **through the reference week, at its first-reported (unrevised) values**.

**Worked example** (national, `reference_date = 2026-01-03`):

| week ending | value the forecaster had (as of the cutoff) | season-final value (scoring truth) |
|---|---|---|
| `2025-12-27` | `<TBD>` | `<TBD>` |
| `2026-01-03` | `<TBD>` (first reported `2026-01-09`) | `<TBD>` |

> **To be filled in from the actual pull.** Run `scripts/get_target_data.R` and read the two values off the generated files rather than transcribing them from anywhere else. Pick a reference date near the season peak, where backfill is largest and the teaching point lands hardest.

### Two target-data files, two different "as of" choices

- **[`target-data/time-series.csv`](target-data/time-series.csv), the versioned observed series (what the forecaster saw).** It carries an **`as_of`** column equal to the forecast `reference_date`. For each `as_of` we record every week from the start of the series up to that date, each at its **latest release on or before `reference_date + 9`** (the cutoff above). This is the series the [`predtimechart`](https://docs.hubverse.io/en/latest/user-guide/dashboards.html) dashboard displays alongside a forecast, so the observed line matches the data the model launched from, including visible backfill. **The full history is repeated in every `as_of` snapshot on purpose:** predtimechart will not render a snapshot that is missing early weeks.

- **[`target-data/oracle-output.csv`](target-data/oracle-output.csv), the scoring truth (what actually happened).** Forecasts are scored against the **season-final** value of each week: the latest release on or before a fixed post-season freeze date set in `scripts/get_target_data.R`. The freeze sits far enough past the end of the season that ordinary backfill has settled (revisions generally complete within about two months), while keeping scores **reproducible**. Scoring against the all-time-latest data would mean that re-running the pipeline months later could silently change past scores.

### Common pitfalls when regenerating

- Do **not** use finalized (all-time-latest) data for the observed `time-series`: it overstates what forecasters knew and makes the observed line disagree with the forecast's starting point at the reference week.
- Do **not** use `release_date <= reference_date` (with no `+9`): the reference week is not published until about `reference_date + 6`, so it would be missing and fall back to a finalized value, which is the exact bug this design avoids.
- Do **not** mix the preliminary and finalized NHSN signals within a single snapshot. This hub uses the finalized signal throughout; the preliminary Wednesday release is a different series with a different revision profile.
- Do **not** score against the latest data: use the frozen post-season snapshot, or a re-run months from now will not reproduce today's scores.

## Evaluation

Forecasts are scored with [`scoringutils`](https://epiforecasts.io/scoringutils/) against `target-data/oracle-output.csv`. Reported metrics are the weighted interval score (WIS), absolute error of the median, and 95% prediction interval coverage.

Two things to know before reading the scores:

- `scoringutils::score()` returns a per-forecast `ae_median`. **MAE only exists after averaging** across forecasts, so a by-horizon or by-model summary is where MAE appears, not the raw score output.
- `scoringutils` does **not** compute 95% interval coverage by default for 0.025/0.975 quantiles. It must be requested explicitly.

## Acknowledgments

This repository follows the guidelines and standards outlined by [the hubverse](https://hubverse.io), which provides a set of data formats and open source tools for modeling hubs.

This repository follows the guidelines and standards outlined by [the hubverse](https://hubverse.io), which provides a set of data formats and open source tools for modeling hubs.
