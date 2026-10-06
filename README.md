# Data Quality Analyzer

Python CLI that ingests a CSV, profiles it, flags quality issues, computes
statistics, generates diagnostic plots, cleans the data, and emits a
Markdown report + cleaned CSV.

## Architecture

Six independent modules, each a single-responsibility class, wired by
`pipeline.py`. No module imports another except through the data it returns
(DataFrame / dataclasses) — this is the seam that lets 6 people work in
parallel without merge conflicts.

```
data_quality_analyzer/
  data_loader.py          Module 1 — CSV ingestion, encoding fallback
  data_profiler.py        Module 2 — per-column structural profile
  quality_checker.py      Module 3 — issue detection (duplicates, nulls, outliers, consistency)
  statistical_analyzer.py Module 4 — descriptive stats, correlation
  visualization_engine.py Module 5 — matplotlib/seaborn plots -> PNG
  cleaning_report.py      Module 6 — cleaning pipeline + Markdown report
  pipeline.py             Orchestrator — instantiates and sequences 1-6
main.py                   CLI
```

### Suggested module ownership (team of 6)

| Module | Owns | Depends on (interface only) |
|---|---|---|
| 1. Data Loader | `LoadResult` contract | none |
| 2. Data Profiler | `ColumnProfile` contract | `LoadResult.dataframe` |
| 3. Quality Checker | `Issue` contract | raw DataFrame |
| 4. Statistical Analyzer | stats/correlation DataFrames | raw DataFrame |
| 5. Visualization Engine | PNG outputs | raw DataFrame + Module 4's `corr` |
| 6. Cleaning & Report | cleaned CSV + `report.md` | outputs of 1-5 |

Integration point is `pipeline.py`: each owner implements against the
dataclass/DataFrame contract their module returns, not against another
module's internals. Agree on the 5 contracts above before writing code —
that's the actual coordination cost for a 6-person team, not the algorithms.

## Setup

```bash
pip install -r requirements.txt
```

## Run

```bash
python main.py --input data.csv --output-dir output/ \
    --missing-strategy median --cap-outliers
```

Outputs land in `output/`: `cleaned.csv`, `report.md`, and PNG plots
(`missingness_heatmap.png`, `distributions.png`, `boxplots.png`,
`correlation_heatmap.png`, `categorical_bars.png` — each only generated if
relevant, e.g. no missingness plot if there are no nulls).

### CLI flags

| Flag | Default | Effect |
|---|---|---|
| `--missing-strategy` | `median` | `median`/`mean`/`mode` impute, `drop_rows`, or `none` |
| `--no-drop-duplicates` | off | keep duplicate rows |
| `--standardize-case` | off | lowercase all text columns |
| `--drop-constant-columns` | off | drop columns with <=1 unique value |
| `--cap-outliers` | off | winsorize numeric columns at ±3σ |

## Known limitations (fix before calling this production-ready)

- Outlier detection is z-score only — breaks on skewed/non-normal
  distributions. Add IQR-based detection as a second method and let the
  caller pick, or auto-select based on a normality test.
- No schema/type-contract validation (e.g. "age must be 0-120",
  "email must match regex"). Currently purely statistical, not semantic.
- `pd.read_csv(..., engine="python", sep=None)` delimiter-sniffing is slow
  on large files — fine for the CSV sizes typical of a course project, not
  for GB-scale ingestion. Swap to explicit delimiter + `engine="c"` if you
  scale this up.
- No streaming/chunked read — full file loads into memory. Bounded by
  `--input` file size vs. available RAM.
- `report.md` embeds relative image links; if you move `cleaned.csv`/plots
  out of `output/` without the report, links break. Convert to a
  self-contained HTML report (inline base64 images) if this needs to be
  shipped as a single deliverable.

## Contributors
1. Dhankecha Ashish
2. Bhikadiya Meet
