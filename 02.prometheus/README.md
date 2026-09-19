# prometheus-rs

A Prometheus-compatible metrics monitoring and alerting system, written from
scratch in Rust with no Prometheus code involved. It scrapes targets over HTTP,
stores the samples in a compressed time series database, answers PromQL,
evaluates recording and alerting rules, and serves the Prometheus HTTP API plus
a self-contained web UI.

```
                                        ┌──▶ HTTP API ──▶ web UI / Grafana
targets ──scrape──▶ TSDB ──PromQL──▶────┤
                     ▲                  └──▶ desktop console
                     └──rules───────────────────alerts──▶ Alertmanager
```

The server and the desktop console drive the same runtime (`src/server.rs`), so
scraping, rule evaluation and reloading behave identically in both.

## Screenshots

![The graph page](screenshots/web-graph.png)

![The desktop console](screenshots/desktop-console.png)

*The web UI answering `scrape_duration_seconds` while scraping itself and an
Elixir/Phoenix application on `localhost:4000`, and the desktop console running
the same runtime in-process. The targets, rules, TSDB and status pages, a
20-slide deck, a 15-page overview document and a Korean README are in
[`docs/`](README.md) ([한국어](README.ko.md)).*

## Quick start

```bash
cargo build --release

# Validate before you run it.
./target/release/promtool-rs check config prometheus.yml

# Run it. The shipped configuration scrapes the server itself.
./target/release/prometheus-rs --config.file=prometheus.yml
```

Or run the desktop console, which starts the same server in-process:

```bash
./target/release/prometheus-rs-gui --config.file=prometheus.yml
```

`gui-demo.yml` is a second sample configuration for looking at the console with
something happening in it: a two-second scrape interval and one target that is
deliberately not there, so the Targets tab has a down target and the
`TargetDown` rule has something to fire on.

All three are also available as one executable — see [One file](#one-file).

The console keeps its HTTP port closed until you open it from the Server tab —
the API has no authentication, and a desktop application should not put one on
the network without being asked. Pass `--web.enable` to open it at startup.

Then open <http://localhost:9090/graph> and try:

| Query | Shows |
| --- | --- |
| `up` | which targets are reachable |
| `rate(prometheus_engine_queries_total[5m])` | queries per second this server is answering |
| `sum by (job) (up)` | reachable targets per job |
| `prometheus_tsdb_head_series` | how many series are in memory |
| `ALERTS` | every pending and firing alert |

## One file

The three binaries are the right shape for a deployment, where the server is the
only thing that belongs on the host. They are the wrong shape for handing
someone a copy: three files that have to stay the same version, two of which
look useless on their own. There is a fourth target that is all three at once.

```bash
# Build it and copy it out under the name it ships as.
powershell -Command "& { Set-Location -LiteralPath '.'; .\tools\bundle.ps1 }; exit $LASTEXITCODE"

dist/prometheus-rs.exe                                  # opens the console
dist/prometheus-rs.exe serve --config.file=prometheus.yml
dist/prometheus-rs.exe promtool check config prometheus.yml
dist/prometheus-rs.exe licenses --write ./notices
```

Running it with no subcommand opens the console, because that is what a double
click should do. An unknown subcommand is an error rather than a guess: reading
`prometheus-rs srve` as a request for the window would start a process on a
headless host that scraped nothing and said nothing about why.

Every arm parses the flags the corresponding binary parses and calls the
function it calls — `src/cli/serve.rs`, `src/cli/promtool.rs` and `gui/src/cli.rs`
are library modules precisely so that there is one copy of each. The only thing
`gui/src/bin/bundle.rs` decides is which of them runs.

The licences travel **inside** it. `NOTICE` requires `THIRD-PARTY-NOTICES.md` to
accompany any binary built from this source, and a single file has no folder to
put it in, so the bundle compiles all three texts in and hands them back:

```bash
prometheus-rs licenses                  # this project's notice
prometheus-rs licenses --license        # the Apache License 2.0
prometheus-rs licenses --third-party    # every linked crate's licence, in full
prometheus-rs licenses --write DIR      # all three as files, for redistribution
```

That is what the extra 3 MB over the console binary is. `--write` reproduces the
files byte for byte, which is what a redistributor needs.

The Cargo target is named `prometheus-rs-bundle` rather than `prometheus-rs`
because the server binary already has that name and two targets in one
workspace cannot write the same file into `target/release`; `tools/bundle.ps1`
copies it to `dist/prometheus-rs.exe`. The help text is pinned to
`prometheus-rs`, so what it calls itself does not depend on the file name.

The window and the terminal share one process, so a console window opens
alongside the GUI on Windows. That is deliberate: `serve` and `promtool` write
to stdout, and a binary that is sometimes a command line tool has to keep one.

## What is implemented

**Storage.** Two tiers. The *head* is in memory with Gorilla compression:
delta-of-delta timestamps and XOR-encoded values, cut into 120-sample chunks,
with series found through an inverted index over label pairs. Durability is a
write-ahead log with per-record CRC32 plus periodic head snapshots, so a restart
replays only what happened since the last snapshot. The log is buffered in this
process and pushed to the disk every `--storage.tsdb.wal-flush-interval`, so
what an unclean stop costs is at most the writes of that interval — with the
flush turned off it is everything since the last snapshot, which is why it is
on by default. A clean stop flushes and snapshots before exiting, and "clean"
covers every way an operating system asks a process to go: SIGTERM and SIGINT
on Unix, and on Windows Ctrl-C, Ctrl-Break, the console window being closed,
a system shutdown and a logoff — each of which is a separate signal there, and
all but the first of which used to skip the flush entirely. Draining is bounded
from outside the serve loop, because closing a console window gives a process
about five seconds in total and one client holding a connection open would
otherwise spend all of them. Out-of-order and
conflicting-duplicate appends
are rejected rather than silently reordered, and staleness markers end a series
instead of letting the last value be carried forward.

Once a block range has elapsed the head is *cut*: its chunks older than the
boundary are written into an immutable block on disk and dropped from memory.
A block keeps its series labels in RAM and its samples on disk, so lengthening
retention costs disk rather than memory. Queries read the head and whichever
blocks overlap the window, and a series spanning the boundary comes back as one
series. Blocks are merged as they accumulate, which keeps the number a query has
to consider small and re-compresses a series whose samples were split across
several of them. Chunks are started fresh at every block boundary so that a cut
moves everything below it — without that, a block range shorter than a chunk's
own span would move almost nothing and the tiering would appear to work while
holding everything in memory.

Measured compression, on scrape intervals with realistic jitter, is about 3.4
bytes per sample for a counter and about 7.2 for a gauge whose value wanders,
against the 16 bytes an uncompressed `(i64, f64)` pair costs. A series whose
value does not change costs well under a byte per sample. Note that a series
also carries a fixed per-series overhead for its label set, so a store holding
many series with few samples each will show a much higher figure in
`promtool-rs tsdb stats` — 2,000 series with four samples each measures at
about 39 bytes per sample, nearly all of it label storage.

`prometheus-rs-qa --bench` measures all of this on your own machine rather than
asking you to take these numbers on trust.

### What retention costs

Measured: a series costs about **310 bytes of memory per block it appears in**,
and about **3 bytes per sample on disk**. Samples in a block are not in memory
at all, so the figure that used to decide capacity — bytes per sample times the
retention window — no longer does.

What is left is the block count, and compaction keeps that small: merging every
three blocks of a level into one turns the 180 two-hour ranges of a fifteen-day
window into a handful. A series present throughout a fifteen-day retention
costs single-digit kilobytes of RAM rather than the several hundred it would
cost held in the head, and the samples themselves cost disk.

Two things still bound you. The head holds one block range at a time, so a very
large cardinality at a two-hour range is still a real amount of memory — shorten
`--storage.tsdb.block-range` if that bites. And a block's index is per block, so
a series that exists for one scrape and never again still costs its labels in
every block it touched until retention removes them.

**PromQL.** A full lexer, parser and evaluation engine:

- instant, range and subquery selectors, with `offset` and `@ start()/end()`
- all binary operators, including `on`/`ignoring`, `group_left`/`group_right`
  and the `bool` modifier
- twelve aggregation operators (`sum`, `topk`, `quantile`, `count_values`, …)
  with `by`/`without`
- 68 functions: `rate`, `irate`, `increase`, `delta`, `deriv`,
  `predict_linear`, `holt_winters`, the `*_over_time` family,
  `histogram_quantile`, `label_replace`, `label_join`, the maths and calendar
  functions, and more
- the semantics that actually decide what a dashboard shows: 5-minute lookback,
  left-open range windows, counter-reset correction, and `rate()`'s
  extrapolation to the window edges (clamped so a counter is never extrapolated
  below zero)

Queries are bounded by a sample budget, a step-count limit and a wall-clock
timeout, and by how many may run at once. Evaluation is CPU-bound, so it runs
on a blocking thread pool rather than on the async runtime the scrape, rule and
remote-write loops share — otherwise a wall of dashboard panels stops the
server collecting anything, at exactly the moment someone is looking at a
dashboard because something is wrong. `--query.max-concurrency` bounds how many
evaluate together; the rest wait.

**Scraping.** Four discovery mechanisms — `static_configs`, `file_sd_configs`,
`http_sd_configs` (the same JSON fetched from a URL, so anything that can render
your inventory becomes a backend) and `dns_sd_configs` (`A` and `AAAA` lookups,
re-resolved on their own interval) — per-job intervals and
timeouts, basic-auth and bearer tokens, `honor_labels`, `honor_timestamps`,
sample and body-size limits, and the full relabelling engine (`replace`, `keep`,
`drop`, `hashmod`, `labelmap`, `labeldrop`, `labelkeep`, `lowercase`,
`uppercase`, `keepequal`, `dropequal`) for both targets and individual samples.
Every scrape produces `up`, `scrape_duration_seconds`,
`scrape_samples_scraped`, `scrape_samples_post_metric_relabeling` and
`scrape_series_added`, whether it succeeded or not.

A job scraping `https` can say what it trusts. `tls_config` takes a `ca_file`
— which *replaces* the public roots rather than joining them, so a job pointed
at an internal authority does not also accept a certificate from a public one —
plus `cert_file` and `key_file` for mutual TLS and, as a last resort with a
loud name, `insecure_skip_verify`. `proxy_url` sends the job's scrapes through
a proxy. Without either, jobs share one client and one connection pool.

`scrape_duration_seconds` covers the whole cycle, fetch and ingest together,
which is what makes it comparable to the interval: when it approaches
`scrape_interval`, that target is about to be sampled less often than it was
configured to be. `prometheus_target_scrapes_exceeded_interval_total` counts
the times it already has, and a scrape whose predecessor is still running is
skipped and counted in `prometheus_target_scrapes_skipped_total` rather than
queued behind it.

**Rules and alerting.** Recording rules write back into the TSDB; alerting rules
run the `Inactive → Pending → Firing` state machine with `for` and
`keep_firing_for`, publish `ALERTS` and `ALERTS_FOR_STATE` series, render
annotations from a template subset, and push to Alertmanager's v2 API with
resends and resolution notices.

**HTTP API.** The Prometheus endpoints, with the same JSON envelope and the same
string-encoded sample values, so Grafana and existing tooling work unchanged:
`/api/v1/query`, `/query_range`, `/series`, `/labels`,
`/label/{name}/values`, `/targets`, `/rules`, `/alerts`, `/alertmanagers`,
`/metadata`, `/format_query`, `/parse_query`, `/status/*`,
`/admin/tsdb/snapshot`, plus `/metrics`, `/-/healthy`, `/-/ready` and
`/-/reload`.

**Web UI.** One HTML document with no external dependencies — no CDN, no
bundler, no build step. Graph page with a canvas chart, metric-name
autocomplete and shareable URLs; targets, alerts, rules, TSDB cardinality,
runtime status and the running configuration. A monitoring UI that needs the
internet to render is useless exactly when you need it.

**Remote write and read.** The server accepts `POST /api/v1/write`, so any
Prometheus can be pointed at it and this becomes its storage backend; it
forwards a copy of what it holds to any number of endpoints, so a short local
window for alerting can sit in front of something built for years; and it
answers `POST /api/v1/read`, so what was written is reachable again — a
Prometheus with a `remote_read` block pointed here sees this server's history as
though it were local. The wire format is the specified
`snappy(protobuf(…))`, encoded by hand rather than by a code generator. The
sender attaches *its own* external labels, which is what lets a store collecting
from ten servers tell them apart.

Each endpoint's position is written to `remote_write_watermarks.json` in the
storage directory after every send and carried across reloads, so a
configuration change or a restart resumes where the forwarding had reached. It
used to restart at the present instant, which meant that every reload — and a
reload happens whenever a target or a rule changes — skipped whatever had been
collected but not yet shipped, permanently, since the position never moves
backwards. An endpoint that was not there before still starts at now: a server
that has been up for a fortnight and is then pointed at a remote store should
not open by shipping a fortnight of history.

Two limits worth stating. Only the `SAMPLES` response encoding is implemented;
a querier that accepts *only* streamed chunks is told so rather than handed a
body it cannot read. And the query engine does not fan out to remote endpoints —
a network hop inside rule evaluation would slow every rule to the speed of the
slowest endpoint. `promtool-rs remote-read` is the way this process reads from
somewhere else.

**Federation.** `/federate?match[]=…` serves the latest value of each selected
series in the exposition format, with each sample carrying its own timestamp, so
a higher-level server scraping it with `honor_labels: true` stores what was
taken here rather than what it happened to ask for. Federating everything is not
offered: at least one `match[]` is required.

**High availability.** The standard pattern rather than a cluster: run two
servers scraping the same targets, give each a different `replica` external
label, and point both at one reader. `--query.replica-label=replica` then makes
a query return one series per replica set with the label stripped. The replica
with more of the window wins and is used *whole* — merging the two sample by
sample would double the density and make `rate()` report a rate neither replica
saw.

**TLS and authentication.** `--web.config-file` supplies a certificate and
credentials. Passwords are bcrypt, in the format `htpasswd -B` writes and
`promtool-rs hash-password` will produce for you; bearer tokens are compared in
constant time; TLS is rustls with nothing older than 1.2. Individual paths can
be exempted so a load balancer can probe `/-/healthy` without a secret, and the
exemption is exact rather than a prefix. Without a web configuration the API is
plain HTTP with no credentials, and the server says so at start-up.

A verdict is remembered for five minutes, and the verification runs on a
blocking thread under a small concurrency cap. bcrypt is deliberately slow — a
cost-12 hash is about 175 ms of one core — and paying that on every request had
two costs. A dashboard polling a few times a second spent half a core on
password checking; and because a rejection cost exactly as much as an
acceptance, anyone who could reach the port could take most of the machine by
sending wrong passwords, including stalling an *exempt* health probe, since the
hashing ran on the same runtime as everything else. Rejections are cached for
the same reason acceptances are: the attack is repetition.

**Desktop console.** `prometheus-rs-gui` runs the whole server in-process and
reads target, rule and storage state straight out of the shared handles, so
there is no serialisation round trip between what is stored and what is on
screen. Eight tabs: a PromQL graph with metric-name autocompletion and a plot,
targets, alerts grouped by name, rule groups with the fraction of their interval
each one consumes, storage size and cardinality, the running configuration with
a reload button, the process's own metrics, and the HTTP listener with a live
log. Queries run on a worker rather than the UI thread, so a query that hits its
two-minute budget leaves the window responsive.

**Self-monitoring.** The server exposes its own metrics in the exposition
format and, by default, scrapes itself. The desktop console's queries are
counted in the same counters as the API's, so
`prometheus_engine_queries_total` describes the process rather than one of its
front ends.

## What is not implemented

Stated plainly, because the gaps matter more than the features:

- **No clustering.** One node stores one copy. Redundancy comes from running a
  second server and deduplicating at the reader, not from replication inside
  this one — so a host loss still costs whatever that host had not forwarded.
- **The query engine reads only local storage.** It serves remote reads and can
  perform one through `promtool-rs`, but a PromQL query here never fans out to
  another server.
- **Only the `SAMPLES` remote read encoding**, not streamed chunks.
- **No exemplars, no native histograms** (classic `_bucket` histograms work),
  no OpenMetrics `_created` handling.
- **Discovery is static, file, HTTP and DNS.** No Kubernetes, EC2 or Consul
  provider — point an `http_sd` endpoint at whatever can enumerate them.
  `SRV` records are refused rather than silently returning nothing, because the
  platform resolver cannot make that query.
- **Alert silencing and grouping live in Alertmanager**, not here.
- **Annotation templates are a documented subset** of Go's `text/template`:
  `$value`, `$labels.x`, `$externalLabels.x`, `printf`, `humanize`,
  `humanize1024`, `humanizePercentage` and `humanizeDuration`. Anything else is
  left in place and reported as a warning rather than silently dropped.
- **No multi-tenancy.** One process serves one tenant's data to everyone who
  can authenticate. There is no per-tenant isolation, quota or accounting.
- **A configuration written for upstream Prometheus will not load unchanged.**
  Unknown fields are refused rather than ignored, which is the right trade for
  a typo and the wrong one for a migration: `kubernetes_sd_configs` and
  friends fail the load with a message naming what *is* supported.
- A metric name beginning with a colon must be written `{__name__=":foo"}`,
  because a leading colon is the subquery separator in PromQL.

One deliberate divergence from upstream: `histogram_quantile(0, ...)` over a
histogram whose first bucket is empty returns that bucket's upper bound, where
upstream divides by the empty bucket's zero count and returns `NaN`. An empty
first bucket says no observation was at or below its bound, so the bound is the
answer and zero — what the arithmetic would otherwise give — is a floor the data
contradicts.

## Configuration

`prometheus.yml` follows the upstream schema. Unknown fields are rejected, so a
typo fails the load instead of being ignored:

```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
  external_labels:
    monitor: prometheus-rs

rule_files:
  - rules/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["localhost:9093"]

scrape_configs:
  - job_name: node
    scrape_interval: 30s
    static_configs:
      - targets: ["localhost:9100"]
        labels:
          env: prod
    relabel_configs:
      - source_labels: [__address__]
        regex: "(.*):.*"
        target_label: instance
        replacement: "$1"
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: "go_gc_.*"
        action: drop
```

Reload without restarting — the endpoint has to be switched on first:

```bash
prometheus-rs --config.file=prometheus.yml --web.enable-lifecycle
curl -X POST http://localhost:9090/-/reload
```

`--web.enable-lifecycle` and `--web.enable-admin-api` are off by default. Both
write: one re-reads every rule file and rebuilds every remote-write forwarder,
the other puts a snapshot on disk. Without `--web.config-file` the API has no
credentials at all, so serving them unconditionally would have meant anything
that could reach the port could trigger them.

A reload that fails leaves the running configuration untouched — a typo in a
rule file cannot take down a working server. A reload that succeeds does not
disturb remote write either: each endpoint keeps the position it had reached
rather than restarting from the present instant, so changing a target does not
leave a gap in whatever this server is forwarding to.

### Flags

| Flag | Default | Meaning |
| --- | --- | --- |
| `--config.file` | `prometheus.yml` | configuration path |
| `--web.listen-address` | `0.0.0.0:9090` | API and UI address |
| `--web.external-url` | derived | base URL used in alert links |
| `--storage.tsdb.path` | `data` | WAL and snapshot directory; empty means memory only |
| `--storage.tsdb.retention.time` | `15d` | how long samples are kept |
| `--storage.tsdb.snapshot-interval` | `2h` | how often the head is snapshotted |
| `--storage.tsdb.wal-flush-interval` | `10s` | how often the WAL reaches the disk; bounds what a kill loses |
| `--storage.tsdb.retention.size` | `0` | disk budget for the blocks, e.g. `50GB`; `0` is no limit |
| `--storage.tsdb.max-series` | `5000000` | series held in memory before new ones are refused |
| `--storage.tsdb.block-range` | `2h` | how much time one persistent block covers |
| `--storage.tsdb.no-blocks` | off | keep every sample in memory instead |
| `--storage.tsdb.compaction-group` | `3` | blocks of one level merged into the next |
| `--web.config-file` | none | TLS certificate and credentials for the API |
| `--query.replica-label` | none | label distinguishing replicas; enables deduplication |
| `--query.lookback-delta` | `5m` | how far back an instant selector looks |
| `--query.max-samples` | `50000000` | per-query sample budget |
| `--query.timeout` | `2m` | per-query wall-clock budget |
| `--query.max-concurrency` | `20` | queries evaluated at once; the rest wait |
| `--scrape.max-concurrency` | `64` | scrapes in flight at once |
| `--web.enable-lifecycle` | off | serve `POST /-/reload` |
| `--web.enable-admin-api` | off | serve the `/api/v1/admin` endpoints |
| `--log.level` | `info` | `error`, `warn`, `info`, `debug`, `trace` |
| `--config.write-default` | off | write a starter config if none exists |

## promtool-rs

Everything works offline, so a bad rule file is rejected in CI rather than at
three in the morning:

```bash
promtool-rs check config prometheus.yml       # config plus every rule file it references
promtool-rs check rules rules/*.yml
cat metrics.txt | promtool-rs check metrics   # lint exposition output
promtool-rs format 'sum by(job)(rate(x[5m]))' # canonicalise an expression
promtool-rs check-web web.yml                  # TLS files, password hashes, exempt paths
promtool-rs hash-password                     # a bcrypt hash for basic_auth_users
promtool-rs tsdb stats data                   # head, blocks and a cardinality report
promtool-rs tsdb blocks data                  # what is on disk, and at which level
promtool-rs remote-read http://host/api/v1/read 'up{job="node"}'
promtool-rs tsdb series data --match 'up{job="node"}'
promtool-rs query instant data 'sum(up)'
promtool-rs query range data 'rate(x[5m])' --start 1700000000 --end 1700003600 --step 1m
```

The `query` and `tsdb` subcommands open the directory directly; point them at a
copy, or at a stopped server's data, rather than at a directory a running server
is writing.

## Layout

| Path | What lives there |
| --- | --- |
| `src/labels.rs` | label sets, matchers, series fingerprints |
| `src/exposition.rs` | the text exposition format, parser and encoder |
| `src/tsdb/chunk.rs` | Gorilla compression |
| `src/tsdb/index.rs` | the inverted index |
| `src/tsdb/wal.rs` | write-ahead log and snapshots |
| `src/tsdb/block.rs` | the persistent tier: immutable on-disk blocks |
| `src/tsdb/codec.rs` | the primitives the on-disk formats share |
| `src/tsdb/mod.rs` | the head block, appends, queries, retention |
| `src/promql/` | lexer, parser, AST, functions, evaluation engine |
| `src/config.rs` | configuration schema and the relabelling engine |
| `src/scrape/` | service discovery and the scrape pipeline |
| `src/rules/` | recording and alerting rules, annotation templates |
| `src/alerting.rs` | the Alertmanager client |
| `src/api/` | the HTTP API and the embedded web UI |
| `src/api/auth.rs` | credentials, and the middleware that enforces them |
| `src/api/tls.rs` | serving the API over TLS |
| `src/remote/` | remote write and read: the wire format and both ends |
| `src/promql/dedup.rs` | choosing one replica of a series at query time |
| `src/server.rs` | the runtime both front ends drive: loops, reload, shutdown |
| `src/cli/` | the server and promtool command lines, as library modules |
| `src/selfmetrics.rs` | the server's own instrumentation |
| `gui/` | the desktop console |
| `gui/src/bin/bundle.rs` | the single-file build: which of the three runs |
| `qa/` | the quality gate and the benchmark |

## Tests and the quality gate

```bash
cargo test                                   # 803 tests
cargo run --release -p prometheus-rs-qa      # 6,603 behavioural checks
node tools/console-tests.js                  # 104 checks over the web console's script
cargo run --release -p prometheus-rs-qa -- --bench
```

803 tests: unit tests next to the code they cover, plus an integration suite
that scrapes a real HTTP target over a real socket, queries the result, fires an
alert on an unreachable target, and restarts a storage through its snapshot and
WAL.

On top of those, `prometheus-rs-qa` runs about six and a half thousand
behavioural checks across twenty-five suites — label sets and fingerprints, matchers, durations, value
formatting, Gorilla compression over a matrix of series shapes, the storage, the
PromQL parser and its round trip, the function registry, an evaluation
conformance table whose expected values are worked out by hand, configuration
and rule validation, relabelling, the exposition format, the scrape assembly,
the HTTP API over a real socket, the persistent blocks, the credential grid, both remote wire formats, replica
deduplication, and the console's own logic.

Two things about it are worth knowing. It fails if fewer checks run than
`--min-checks` (2,000 by default), because a suite that quietly stops running is
the one failure a green gate cannot show you. And it fails if two checks share a
name, because that is a loop missing its index and the symptom is a failure
nobody can locate.

`--bench` measures compression, memory per series, ingest and replay rates and
query latency against cardinality, so the numbers in this file can be checked
rather than believed.

The web console is about a thousand lines of JavaScript inside a Rust string
constant, and no Rust test can execute a line of it. `tools/console-tests.js`
does: it pulls the page's pure functions out of `src/api/web.rs` — extracted,
not copied, since a copy would pass while the real one was broken — and runs
them. It was written after three defects were found living in that gap: a range
picker that read `1y` as sixty seconds, a status page that threw on a timestamp
it could not format, and a grouping that fell over a scrape job named
`constructor`. Node is a build requirement for it in the same way python already
is for the licence notices.

## Licence

Apache-2.0. The full text is in `LICENSE`, and `NOTICE` carries the attribution
that Apache-2.0 section 4(d) asks to be passed on.

The binaries statically link 487 crates, every one of them permissively
licensed — there is no copyleft in the tree — but permissive is not the same as
unconditional: 112 of them are MIT-only, and MIT asks for its copyright and
permission notice to accompany all copies. `THIRD-PARTY-NOTICES.md` carries
those notices and must travel with any binary built from this source. The
single-file build carries them compiled in, and `prometheus-rs licenses --write`
writes all three texts back out byte for byte — see [One file](#one-file).

That file is generated, not maintained:

```bash
python tools/third-party-notices.py           # regenerate
python tools/third-party-notices.py --check   # fail if a dependency changed without it
```

`ci.ps1` runs the `--check` form, because a dependency added without
regenerating the notices is how a binary ends up shipping without the
attribution its licences require.
