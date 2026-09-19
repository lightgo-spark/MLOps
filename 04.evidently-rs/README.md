# evidently-rs 0.3

A Rust reimplementation of the Evidently AI monitoring stack: a **library**, a **CLI**, a
**desktop console** and a **local web console**. The statistics are implemented from scratch — no
external stats crate — including the special functions they need (lnΓ, regularised gamma/beta, the
Kolmogorov distribution, Bessel K). 47 unit tests pin the engine to published critical values
(χ²=3.8415 → p .05, t=2.228 at df 10 → .05, Fisher's tea-tasting table → .4857, CvM 0.461 → .05)
and to regression invariants.

```
cargo build --release                      # library + evidently CLI
cargo build --release --features gui       # + evidently-studio desktop console
```

---

## evidently-studio — the desktop console

`evidently-studio` is a local monitoring console: load a reference and a current dataset, run the
analysis, and drill into whichever column moved. `evidently-studio --serve` (or `evidently serve`)
puts the same report behind a browser at `http://127.0.0.1:4000/` — a local, read-only web console
with the HTML report, the test suite and a JSON API. Nothing leaves the machine and there is no
account to create.

```
evidently-studio                       # empty console
evidently-studio ref.csv cur.csv       # open two CSVs as reference / current
evidently-studio --demo                # generated data with the analysis already run
evidently-studio --demo --page drift   # open on a specific page
evidently-studio --serve --open        # web console at 127.0.0.1:4000 instead of the window
```

![Overview](docs/screenshots/overview.png)

**Overview** — the screen you check first: dataset drift verdict, drifted share against its threshold,
missing and duplicate rates, model metrics, and the most-drifted columns ranked side by side with the
distribution behind the top one.

![Drift](docs/screenshots/drift.png)

**Drift** — every column with the test that was actually run, its score, its threshold and the verdict.
Selecting a row shows the reference-versus-current distribution that justifies it, because a drift
score without its distribution is a number nobody can act on. The test can be overridden per data type
and recomputed in place.

![Model performance](docs/screenshots/model.png)

**Model performance** — confusion matrix, per-class precision/recall/F1, ROC and calibration curves for
classification; error distribution, bias and tail statistics for regression.

![Tests](docs/screenshots/tests.png)

**Tests** — the same suite the CLI uses as a CI gate, with each condition, the value it saw and the
verdict. `evidently tests` exits 1 when any test fails, so the console and the pipeline agree.

![Data](docs/screenshots/data.png)

**Data** — CSV loading, column roles and what the report should contain. Also in the console: **Data
quality** (per-column summaries with histograms), **Text evals** (14 descriptors over free-text
columns), **Trends** (metric history across stored snapshots), **Settings** (theme, HTML/JSON export).

Every analysis runs on a worker thread, so the window stays responsive: a full report over 100k rows
takes most of a second (see the performance report), and a console that freezes for that long reads as
broken.

### Local web console

`evidently-studio --serve` — or the same server from the CLI, `evidently serve --demo --open` — puts
the report behind a browser at `http://127.0.0.1:4000/`. It is a standard-library HTTP server: no
Python, no framework, no account, and the payload is rendered once so a request never waits on the
engine. `/` is the index below, `/report` the full HTML report, `/tests` the suite, and
`/api/report.json`, `/api/tests.json`, `/healthz` cover scripts and probes. `--host` / `--port`
change the binding; `--host 0.0.0.0` is the one way to expose it beyond the machine.

![Web console](docs/screenshots/web-console.png)

**Web console index** — the drift verdict, the share of columns that moved and the per-column table,
with the test that was actually run. This is the same report the desktop console shows, in a browser.

![Web report](docs/screenshots/web-report.png)

![Web tests](docs/screenshots/web-tests.png)

**Web report and tests** — the full self-contained HTML report (SVG charts, no external JS/CSS) and
the CI test suite, both readable without the desktop build.

### How this compares to hosted monitoring platforms

Evidently Cloud, Arize, WhyLabs and Fiddler are hosted observability platforms; this is a local tool
built on the same open-source ideas. The differences that matter when choosing:

| | evidently-rs Studio | Hosted platforms (Evidently Cloud, Arize, WhyLabs, Fiddler) |
|---|---|---|
| Deployment | Single executable, no server, no account | SaaS (self-hosted tiers on some plans) |
| Data location | Never leaves the machine | Data or profiles are uploaded to the vendor |
| Cost model | None — Apache-2.0 | Usage-based subscription |
| Drift tests | 13, selectable per column and per type | Comparable coverage, usually fewer knobs exposed in the UI |
| Column drill-down | Yes, with reference/current overlay | Yes — the pattern this console follows |
| Metric history | Local snapshot folder, plotted in Trends | Managed store with retention and dashboards |
| Alerting / scheduling | Not included — use the CLI exit code in a scheduler | Built in (email, Slack, PagerDuty) |
| Multi-user, RBAC, audit | No | Yes |
| LLM tracing, embedding drift | No | Yes on most platforms |
| Scale | One machine; 100k × 10 in ~0.8 s | Distributed, production-traffic volumes |

Choose the console for local analysis, notebooks-replacement work, air-gapped environments and CI
gates; choose a hosted platform when you need alerting, retention, access control and team workflows.

---

## Evidently ↔ evidently-rs

| Evidently | Module | Contents |
|---|---|---|
| `Dataset`, `ColumnMapping` | `column`, `mapping`, `io` | Typed columns, missing values, row filters, CSV type inference |
| `stattests` | `stats` | KS · Wasserstein · PSI · JS · KL · Hellinger · TVD · χ² · z-test · Fisher exact · Welch t · Mann–Whitney · Cramér–von Mises |
| `DataDriftPreset` / `TargetDriftPreset` | `drift` | Evidently's automatic test selection, per-column overrides, rayon parallelism |
| `DataSummaryPreset`, `DatasetCorrelationsMetric` | `quality` | Column summaries, duplicate/constant/empty columns, Pearson and Spearman |
| `ClassificationPreset` | `metrics::classification` | Confusion matrix, P/R/F1 (per class, macro, weighted), ROC-AUC, PR-AUC, log loss, Brier, calibration; binary, multiclass and probabilistic inputs |
| `RegressionPreset` | `metrics::regression` | MAE · MAPE · RMSE · R², error bias and normality (skew, kurtosis), under/over-estimation groups |
| `TextEvals` / descriptors | `text` | 14 descriptors — length, word count, sentence count, OOV share, lexicon sentiment, regex, lexical diversity … — fed back into drift and quality |
| `Report`, `.json()`, `.save_html()` | `report`, `html` | Preset composition, JSON snapshots, a dependency-free single-file HTML report (SVG charts, ROC, confusion heatmap) |
| `TestSuite`, `TestPreset` | `test_suite` | 33 tests, automatic condition generation via `DataStabilityTestPreset` |
| `Workspace` / `Project` | `workspace` | Snapshot storage, listing, metric time series |
| — | `bin/evidently` | CLI: drift / quality / classify / regress / text / tests / serve / snapshots / series |
| — | `bin/evidently-studio` | Desktop console and local web console (feature `gui`, `--serve`) |

## Command line

```bash
evidently drift    --reference ref.csv --current cur.csv --id id --datetime ts --text review --html drift.html --workspace ./ws --project demo
evidently drift    --reference ref.csv --current cur.csv --num-stat-test cramer-von-mises --cat-stat-test psi --bins 20 --exclude noisy_col
evidently drift    --reference ref.csv --current cur.csv --cat-stat-test psi --smoothing 1e-4   # PSI smoothing floor
evidently classify --reference ref.csv --current cur.csv --target y --prediction p --html cls.html
evidently regress  --current cur.csv --target y --prediction yhat --json reg.json
evidently tests    --reference ref.csv --current cur.csv      # exit code 1 when a test fails (CI gate)
evidently serve    --demo --port 4000 --open                  # local web console, no Python needed
evidently serve    --reference ref.csv --current cur.csv --target y --prediction p
evidently series   --workspace ./ws --project demo classification.accuracy
```

`evidently serve` binds `127.0.0.1:4000` by default (`--host` / `--port` to change it) and routes
`/` (index with the drift verdict and a per-column table), `/report`, `/tests`, `/api/report.json`,
`/api/tests.json` and `/healthz`. It is the standard library only: no server framework, no
dependencies, and `--host 0.0.0.0` is the one way to expose it beyond the machine.

```rust
let report = Report::new().with_mapping(ColumnMapping::new().target("y").prediction("p"))
    .with_data_drift(DriftOptions::default()).with_classification().with_data_quality()
    .run(Some(&reference), &current)?;
report.save_html("report.html")?;
```

`cargo run --example basic` runs the whole pipeline end to end.

Exit codes: `0` success, `1` test suite failed (`tests` only), `2` execution error.

## Default drift rules (same as the Evidently documentation)

Numeric n≤1000 → KS (p<.05), n>1000 → Wasserstein/σ (≥.1); categorical n≤1000 → χ² (z-test when
binary, p<.05), n>1000 → Jensen–Shannon (≥.1); dataset drift = share of drifted columns ≥ .5.

Here *n* is the number of **analysable values** — missing and non-finite values (NaN, ±∞) are not
counted.

## Behavioural contracts

These four points differ from Evidently, or are not obvious from its documentation, so they are stated
explicitly.

- **Data drift does not look at target or prediction.** As in Evidently's `DataDriftPreset`, the id,
  datetime, target, prediction, probability and text columns are excluded; target and prediction are
  covered separately by `with_target_drift`. Otherwise the dataset drift share would be contaminated by
  movement in the model's own output.
- **A column that cannot be tested is recorded in `skipped` instead of killing the report.** That covers
  columns with fewer than two values, columns whose type changed between the two datasets, and columns
  for which the requested `--stat-test` does not apply. Every other column still reports, and the reason
  appears in the CLI log and in the HTML report's *Skipped columns* section.
- **NaN and ±∞ are stored distinctly from missing values, and excluded from statistics.** An `inf` or
  `nan` token in a CSV is not the same thing as an empty cell, so it is reported separately through
  `ColumnSummary.n_infinite_or_nan` (the *Inf/NaN* column in the HTML) and kept out of means, quantiles
  and drift tests.
- **The magnitude PSI and KL report is set by the smoothing floor.** Both diverge when a category
  disappears completely, so empty bins get a floor — which makes the reported magnitude a function of
  that floor rather than of the data. For a 50/50 → 100/0 shift, PSI is 3.45 at a floor of 1e-3, 4.60 at
  1e-4 and 6.91 at the default 1e-6. **The verdict (≥0.2) is the same either way, but the number is not**,
  so only compare PSI values from runs that used the same floor. Change it with `--smoothing` or
  `DriftOptions::smoothing`.

A few numerical conventions: when the reference sample is constant there is no scale to normalise by, so
Wasserstein returns `∞` rather than a raw distance (comparing a value in the original units against a 0.1
threshold would be meaningless). An empty sample, or one that is entirely non-finite, yields p=1 — no
evidence. Tail probabilities are computed directly: `erfc` for the normal distribution and the upper
incomplete gamma `gamma_q` for chi-squared, because `1 − Φ(z)` and `1 − P(a,x)` collapse to exactly zero
at |z|≳8.3 and χ²≳75 respectively. A workspace project name must be a single directory name directly
under the workspace root. MAPE is stored as a ratio and displayed as a percentage.

## Performance

Measured on this repository's own harness (`examples/perf.rs`), 95 cases run twice — once on 24 threads,
once with `RAYON_NUM_THREADS=1`. Full write-up, method and raw tables:
[docs/evidently-rs_performance_report.docx](docs/evidently-rs_performance_report.docx).

At 100k rows × 10 columns on an i7-13700F:

| Operation | Median | Throughput |
|---|---|---|
| Full report (drift + quality + classification) | 807 ms | 124k rows/s |
| Dataset drift, 10 columns (24 threads) | 22.6 ms | 4.4M rows/s |
| Data quality summary, 10 columns | 331 ms | 300k rows/s |
| Test suite, 22 tests | 290 ms | 344k rows/s |
| CSV read, 11 columns (17.9 MB) | 101 ms | 178 MB/s |
| HTML report rendering | 23.8 ms | — |

What the measurement showed: no operation is quadratic (log–log slopes 0.9–1.1); histogram-based tests
(PSI, JS, KL, Hellinger) are 3–6× cheaper than sort-based ones (KS, Mann–Whitney, Wasserstein, CvM); the
quality summary — not drift — dominates a report, and it is still single-threaded.

```bash
cargo run --release --example perf -- perf.json
```

## Not implemented

Evidently Cloud / the hosted UI server (a local web console is built in — `evidently serve`),
embedding drift, LLM-as-judge calls, the Spark backend.

## License, attribution and trademarks

Apache-2.0 — see [LICENSE](LICENSE). The code, the demo-data generator, the icon and the screenshots
are original to this repository.

- **Independent project.** evidently-rs is an independent reimplementation inspired by Evidently
  AI's public documentation and API naming. It is not affiliated with, sponsored by or endorsed by
  Evidently AI, Inc. "Evidently", "Evidently AI" and related marks belong to their respective
  owners, and are used here only to describe the project's origin and compatibility (nominative
  use) — no logos or branding of theirs are bundled.
- **No third-party code or assets.** The statistics are implemented from the published mathematical
  definitions (Kolmogorov–Smirnov, Wasserstein, PSI, Jensen–Shannon, KL, Hellinger, TVD, χ², z,
  Welch t, Mann–Whitney, Cramér–von Mises, Fisher exact and the t/χ²/F distributions), and the unit
  tests pin the results to published critical values. No Evidently source code, documentation text,
  screenshots, datasets or branding is copied into this repository.
- **Screenshots** are captures of this software running on synthetic data generated at runtime;
  they contain no real or personal data.
- **Dependencies** (serde, serde_json, thiserror, rayon, csv, clap, chrono, regex and the optional
  eframe/egui/rfd/image stack) stay under their own licenses (MIT and/or Apache-2.0). They are used
  as published crates through Cargo, not vendored or modified.
- **No warranty**, as set out in the Apache-2.0 text. The tool reports statistics, not legal,
  financial or compliance advice.
