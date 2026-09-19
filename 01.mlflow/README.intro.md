# MLflow Studio — a short introduction

**MLflow-compatible experiment tracking as a single Rust binary.** No Python, no
separate server, no container. It can run **without opening a port at all**, and
becomes a REST server when you want one.

> This page is the **short** introduction.
> · Five minutes, hands on → [`QUICKSTART.md`](QUICKSTART.md)
> · Design reasoning, measured numbers, every feature → [`README.ko.md`](README.ko.md) (Korean)
> · Operations and incident handling → [`OPERATIONS.md`](OPERATIONS.md) (Korean)
> · Longer English summary, with the numbers → [`README.en.md`](README.en.md)

![The Charts tab, several runs overlaid](docs/screenshots/charts.png)

## In one paragraph

It reads what your training already leaves behind — an `mlruns/` directory, an
`mlflow.db`, a log file, a CSV — puts it in its own SQLite store, and shows it in
a desktop window and a browser console. Training that runs elsewhere can reach it
over the MLflow tracking REST API, so `mlflow.set_tracking_uri()` is the whole of
the client-side setup. Copying one exe and double-clicking it is the whole of the
install.

## What it does

```
[ purely local ]  no port
  training script ──file:./mlruns──> directory ──watch/incremental──> SQLite ──> UI
  log file ───────parser──────────────────────────────────────────-> SQLite
  CSV / JSON ─────importer────────────────────────────────────────-> SQLite
  by hand ────────editor──────────────────────────────────────────-> SQLite

[ optional network ]  --features server (on by default)
  training elsewhere ──HTTP :5000──> REST API ──> the same SQLite
```

## Why you would use it — three reasons

**1. It starts without Python.** Standing up a tracking server used to mean
standing up a Python environment first. Here it is one exe, with SQLite linked
in statically.

**2. The record is editable.** Most tracking tools are read-only. Here a
mistyped parameter, a wrongly logged metric series, or a run whose clock is off
can be **fixed in the editor, changed in bulk, and undone** — all of it.

**3. You do not have to open a port.** Built with `--no-default-features`, the
network code is not "unused" — it is **not in the binary**. That is a deliberate
property for a tool that sits next to training data.

## Five minutes

```powershell
# Build (Rust 1.92 / edition 2024)
cargo build --release

# Desktop UI
.\mlflow_studio.exe

# Ingest what you already have: a directory is a file store, a file is an mlflow.db
.\mlflow_studio.exe --scan .\mlruns
.\mlflow_studio.exe --scan .\mlflow.db

# Headless REST server (the address your training script points at)
.\mlflow_studio.exe --serve --port 5000
```

One line on the training side:

```python
import mlflow
mlflow.set_tracking_uri("http://127.0.0.1:5000")
```

`--help` is the only complete list of flags: it shows what was actually compiled
into *that* binary, so a build without a feature does not advertise its flags.

## Ways in

| Route | When you use it |
|---|---|
| Watch `mlruns/` | Training already runs against `file:./mlruns`. Ingest is incremental, so rescanning is cheap |
| Read `mlflow.db` | The team's history is already in MLflow's SQLite backend |
| REST API | Training runs on another machine. The MLflow client attaches unchanged |
| CSV / JSON import | The table was produced outside any tracking tool |
| Log parser | The metrics only ever existed in a log file. Rules are written with a live preview |
| Editor | There is little to enter, or something already in the store needs correcting |

## Surfaces

**16 desktop tabs** in the default build (17 in total; the Notifications tab
needs the `notify` feature) — Overview · Experiments · Runs · Run Detail ·
Compare · Charts · Traces · Models · Sources · Editor · Import · Log Parser ·
Export · Server · Access · Settings.

Alongside them, a **browser console**: one link for a colleague, plus read-only
share links.

| | |
|---|---|
| [`docs/screenshots/runs.png`](docs/screenshots/runs.png) | The Runs tab, with filter syntax and dynamic columns |
| [`docs/screenshots/charts.png`](docs/screenshots/charts.png) | Training curves overlaid |
| [`docs/screenshots/console-run.png`](docs/screenshots/console-run.png) | A run page in the web console |
| [`docs/screenshots/overview.png`](docs/screenshots/overview.png) | The Overview tab |

## Boundaries — what it does not do

- **Windows is the supported platform.** The installer is an NSIS package, the
  commands in the documentation are PowerShell, and CI builds and tests on
  Windows only.
- **It sends nothing outward.** No telemetry, no crash reporting, no update
  check. The only outbound call in the binary is to an identity provider, and
  only when `oidc` is compiled in and configured.
- **It is a single-node tool.** A PostgreSQL backend exists for the case where
  two servers must share one store, but scalability in a multi-worker,
  object-store deployment is not an axis this product measures.

## The numbers are deliberately not repeated here

Test counts, gate counts and performance figures have a single source:
[`README.ko.md`](README.ko.md). The gates in `ci.ps1` read those sentences and compare
them against what a run actually produced. An introduction that copied the same
numbers would sit outside that check and go quietly stale — which is a failure
this project has already had, so it is not repeated.

## Licence and notices

[`LICENSE`](LICENSE) — a commercial licence, not an SPDX identifier. Full licence
texts for dependencies are in
[`THIRD-PARTY-LICENSES.md`](THIRD-PARTY-LICENSES.md), the attribution summary is
in [`NOTICE.md`](NOTICE.md), and the vulnerability reporting process is in
[`SECURITY.md`](SECURITY.md).

**MLflow is a trademark of the Linux Foundation.** This product is not
affiliated with, sponsored by or endorsed by the Linux Foundation or the MLflow
project. It is an independent reimplementation of MLflow's tracking API and
contains no MLflow source code. Using the trademark **in the product name
itself** is a separate question from the descriptive use of "MLflow-compatible",
and it is recorded as unresolved in [`LICENSE`](LICENSE).
