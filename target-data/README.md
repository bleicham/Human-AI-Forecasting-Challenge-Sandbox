# Target data

This folder holds the hub's target (truth) data:

- `time-series.csv` — the versioned observed series (with an `as_of` column) the dashboard shows next to forecasts.
- `oracle-output.csv` — the season-final scoring truth forecasts are evaluated against.

**These files are not checked in yet.** They are regenerated from NHSN influenza
hospital-admissions data by [`scripts/get_target_data.R`](../scripts/get_target_data.R).
See the "Data versions, reporting backfill, and forecast timing" section of the
[top-level README](../README.md) for how the `as_of` and season-final values are
defined, and run that script from the hub root to produce both files.

> Note: `scripts/get_target_data.R` currently still pulls ILINet (wILI) data and
> must be updated to fetch the NHSN `wk inc flu hosp` signal before the target
> data will match this hub's `tasks.json`.
