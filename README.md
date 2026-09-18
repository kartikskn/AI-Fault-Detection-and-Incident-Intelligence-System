# Pulse — AI Fault Detection & Incident Intelligence System

An end-to-end incident-intelligence pipeline built on
simulated production telemetry, with an honestly-measured evaluation of
where it works and where it doesn't.

## Overview

Pulse simulates production-style service telemetry with injected faults
(latency spikes, memory leaks, config regressions, and more), detects
unusual behaviour in that telemetry, groups related anomalies into
incidents, ranks the most likely root cause for each incident, collects the
supporting evidence, and presents the whole investigation through a
dashboard — with an LLM used only to narrate the evidence already gathered,
never to decide anything.

It exists to demonstrate an ambiguous problem turned into a well-engineered,
honestly-measured Python system: every number below was produced by a
command you can re-run, and every limitation is stated rather than hidden.

See [docs/milestones.md](docs/milestones.md) for the implementation roadmap
and [progress.md](progress.md) for full session-by-session project history.

## Demo

![Pulse dashboard — incident investigation view](docs/results/plots/dashboard_screenshot.png)

A real, completed `eval_fast` run, viewed through the Streamlit dashboard:
incident navigator on the left, selected incident's timeline, top-3 ranked
root causes, key evidence, and explanation together on one screen. No LLM
provider was configured for this screenshot, so the explanation shown is
Pulse's deterministic, evidence-only fallback — labelled as such in the UI,
not disguised as a model output.

## How It Works

```
fault → telemetry → anomaly detection → incident correlation
      → root-cause ranking → evidence → explanation
```

A scenario config injects one or more faults (e.g. a memory leak on one
service) into a simulated service topology. The simulator produces raw
per-service, per-metric time series. Feature engineering turns those into
rolling statistics with no look-ahead. Four detectors (a random floor, a
static threshold, a robust rolling z-score, and an IsolationForest) flag
anomalous points. Correlation groups related anomalies — across services and
time — into incidents. Root-cause ranking scores each incident's candidate
services against a heuristic signature. A bounded evidence bundle is
assembled from the incident's own facts, and an LLM (or a deterministic
template if none is configured) narrates it.

## Architecture

```mermaid
flowchart TD
    A[Simulated telemetry] --> B[Feature engineering]
    B --> C[Anomaly detection]
    C --> D[Incident correlation]
    D --> E[Severity + root-cause ranking]
    E --> F[Evidence bundle]
    F --> G[(PostgreSQL)]
    F --> H[Grounded LLM]
    G --> I[FastAPI]
    H --> I
    I --> J[Streamlit dashboard]
```

**The deterministic pipeline establishes the incident facts; the LLM is used
only to explain a bounded evidence bundle.** Detection, correlation,
severity and root-cause ranking are pure, seeded functions with no LLM
involvement (docs/architecture.md A4). The LLM never decides any of those —
it only narrates an already-persisted evidence bundle, and its output is
discarded unless it passes grounding validation (A5). FastAPI is the only
read/write surface; Streamlit is presentation only, reached exclusively over
HTTP (§2.1, mechanically checked by `tests/test_architecture.py`).

## Results

All numbers below are measured, not targets (see
[docs/evaluation.md](docs/evaluation.md) for full methodology, seed set, and
every target-vs-measured comparison). Reported over 5 independently-seeded
validation runs (`val.yaml`, eval seeds 2002-2006) unless stated otherwise.
**The held-out test seed has not been touched** — these are validation-seed
numbers, not final frozen-method numbers.

| Metric | Result | Command |
|---|---|---|
| Episode detection rate (B2 / M1, all 6 fault types) | **1.000 ± 0.000** | `python -m pulse.eval.multiseed` |
| Episode detection rate (B1) | 0.667 ± 0.000 — misses `config_regression`/`latency_injection` entirely on every seed | same |
| Incident alert-reduction factor (production correlator) | **27.79× ± 1.34** (14,947 anomalies → 538 incidents, mean) | same |
| Incident purity (IoU≥0.3) | 0.933 ± 0.149 | same |
| False incidents/hour | 5.575/hr — misses the ≤0.5/hr target | same |
| Root-cause top-1, **ORACLE mode** (correlation assumed perfect) | **0.933 ± 0.091** | same |
| Root-cause top-1, **end-to-end mode** (real pipeline, both stages) | **0.267 ± 0.365** | same |
| `(T_window, D_max)` sweep (single seed, 8 of 20 planned cells) | Widening `t_window` 60s→300s and `d_max` 1→4 both improve recall/false-incident-rate/alert-reduction with no purity cost on this seed | `python -m pulse.eval.sweep` |
| Feature engineering wall-clock (`val.yaml`, 12 services, 4 days) | 23.0s | `python -m pulse.bench.suite` |
| Scenario simulation wall-clock | 0.41s | same |

The 0.933 root-cause figure is measured in **ORACLE mode** — ranking is
scored against ground-truth incident membership, isolating the ranker's own
quality from correlation's. It is **not** an end-to-end accuracy number.
The real, end-to-end root-cause top-1 accuracy — running the actual
correlator and then the actual ranker in sequence — is **0.267 ± 0.365**.
The gap between the two (0.933 vs. 0.267) is attributable almost entirely to
incident correlation precision, not to root-cause ranking: **evaluation
identified incident correlation as the main end-to-end bottleneck**, not the
ranker.

## Limitations / Future Work

Stated plainly, not hidden:

- **All telemetry is simulated.** Every metric, fault, and incident in this
  project comes from `pulse.core.sim`, a deterministic synthetic generator —
  not a real production system. The results above measure how well Pulse's
  pipeline recovers faults it itself injected into synthetic data, not
  real-world incident-response performance.
- `noisy_neighbour` (the adversarial two-distant-faults-same-window
  correlation scenario) was attempted twice and **not completed** — the
  single biggest gap in the M10 evaluation.
- The `(T_window, D_max)` correlation-parameter sweep covers 8 of an
  originally-planned 20 grid cells, and does not characterise the widest
  settings at all — the production correlator's exact-connected-components
  recomputation does not scale to those settings at this scenario's anomaly
  volume within the time available (a correctness choice, not a bug).
- The held-out test seed was not touched, by design — every number above is
  a validation-seed number, not a final frozen-method number.
- AI-layer metrics (grounding-validation pass rate, explanation quality)
  were out of M10's scope and were not formally benchmarked.
- The final end-to-end pipeline runtime figure was not freshly re-measured
  in M10; it reuses a prior measurement.
- **Incident correlation quality remains the main technical limitation** —
  it is why end-to-end root-cause accuracy (0.267) trails oracle-mode
  accuracy (0.933) so far.
- Broader load/scaling analysis (DB indexing under load, API load testing, a
  service-count scaling curve) remains future work; it was scoped out of
  M10 for time, not run and hidden.

None of this is presented as production-ready. Pulse is a portfolio project
demonstrating a real, working pipeline with an honest account of where it
currently falls short.

## Tech Stack

- **Simulation & science core:** Python 3.11+, NumPy, pandas, scikit-learn
  (IsolationForest), PyYAML, PyArrow — pure, deterministic, no I/O
  (`pulse.core`).
- **Persistence:** PostgreSQL, SQLAlchemy, Alembic.
- **API:** FastAPI, Pydantic, Uvicorn.
- **Async execution:** Celery, Redis (broker only — no result backend; task
  outcomes live in Postgres).
- **AI explanation layer:** Anthropic Python SDK, with a evidence-grounding
  validator and a deterministic fallback (no external dependency required).
- **Dashboard:** Streamlit, Plotly.
- **Testing/tooling:** pytest, Hypothesis (property tests), ruff, mypy
  (`--strict`).

## Running Locally

### Setup

```sh
make test        # run the test suite
make lint        # ruff check + format check
make typecheck   # mypy
```

Each target creates a local virtual environment (`.venv/`) and installs the
project automatically — no manual setup step is required.

`make test`'s Postgres-backed tests need a real database:
`docker compose up -d db` (or
`docker run -p 5432:5432 -e POSTGRES_USER=pulse -e POSTGRES_PASSWORD=pulse -e POSTGRES_DB=pulse postgres:16`
if the `docker compose` plugin isn't available), then create the dedicated
test database once with
`psql -h localhost -U pulse -d pulse -c "CREATE DATABASE pulse_test;"`.
Its Redis-backed tests similarly need `docker compose up -d redis` (or
`docker run -p 6379:6379 redis:7`). Tests against a database or broker that
isn't reachable are skipped, not failed.

### Running the API and worker

Four processes: Postgres, Redis, the FastAPI app, and a Celery worker.

```sh
docker compose up -d db redis                                       # once
.venv/bin/alembic upgrade head                                      # once, applies the schema
.venv/bin/uvicorn pulse.api.main:app --reload                       # terminal 1: API on :8000
.venv/bin/celery -A pulse.tasks.pipeline worker --loglevel=info     # terminal 2: worker
```

Open `http://localhost:8000/docs` for the interactive OpenAPI UI, or drive it directly:

```sh
# 1. Create and enqueue a run (simulates a 12-hour scenario, then detects,
#    correlates, scores, and ranks it -- on the Celery worker, not the
#    request path). Returns almost immediately with status: "queued".
curl -s -X POST localhost:8000/runs \
  -H 'content-type: application/json' \
  -d '{"scenario_name": "eval_fast", "seed": 1}'
# -> {"run_id": "...", "status": "queued", "n_anomalies": 0, "n_incidents": 0, ...}

# 2. Poll until the worker finishes (typically well under a minute for
#    eval_fast); status moves queued -> running -> completed (or failed).
curl -s localhost:8000/runs/<run_id>
# -> {"run_id": "...", "status": "completed", "n_anomalies": 2536, "n_incidents": 82, ...}

# 3. List its incidents.
curl -s localhost:8000/runs/<run_id>/incidents

# 4. Retrieve one incident's members, severity, and ranked root causes.
curl -s localhost:8000/runs/<run_id>/incidents/<incident_id>

# 5. Get the bounded evidence bundle backing that incident's explanation --
#    what a citation id in step 6's response (e.g. "E3") refers to.
curl -s localhost:8000/runs/<run_id>/incidents/<incident_id>/evidence

# 6. Get an LLM-generated explanation of one incident. Evidence-grounded and
#    grounding-validated (docs/ai-plan.md §3-5); if no provider is
#    configured, or the model's response fails validation after one repair
#    attempt, a deterministic template built only from the same evidence is
#    returned instead -- `status` ("ok" | "repaired" | "fallback" | "failed")
#    and `citations_valid` always say which. Cached by evidence-bundle hash,
#    so repeated requests for an unchanged incident do not re-call the
#    provider.
curl -s localhost:8000/runs/<run_id>/incidents/<incident_id>/explanation

# 7. Get one metric's raw time series for a run -- what the dashboard's
#    timeline plots.
curl -s localhost:8000/runs/<run_id>/metrics/<service>/<metric>
```

`PULSE_DATABASE_URL` (default `postgresql+psycopg://pulse:pulse@localhost:5432/pulse`)
and `PULSE_REDIS_URL` (default `redis://localhost:6379/0`) override the API's
and worker's connection strings — both processes must point at the same
values to work together.

### Configuring the real LLM provider (optional)

**Explanations work with no configuration.** With no `ANTHROPIC_API_KEY`
set, `GET .../explanation` returns the deterministic fallback
(`status: "failed"`, `citations_valid: true`) — Pulse's own facts, narrated
without a model. To use the real Anthropic provider instead, set:

```sh
export ANTHROPIC_API_KEY=sk-...        # required to select the real provider at all
export PULSE_LLM_MODEL=claude-opus-5   # optional, defaults to claude-opus-5
```

Never committed, never logged. CI has no credentials and always exercises
the fake-provider/fallback path (`pulse.llm.fake_provider.FakeProvider`) —
the real provider is exercised only by a manual smoke test.

### Running the dashboard

With the API already up (the steps above), start Streamlit:

```sh
.venv/bin/streamlit run src/pulse/dashboard/app.py
```

Opens on `http://localhost:8501`. One page: pick a run, pick one of its
incidents from a compact list, and see its timeline (metric selector,
anomaly markers, incident window highlighted), top-3 ranked root causes,
key evidence, and the AI explanation together on one screen — each evidence
item that names a metric can jump the timeline straight to it
(docs/product.md FR-10.3). Raw technical detail (evidence IDs, detector
names, lower-ranked candidates) stays available behind compact expanders
rather than crowding the primary view. Reads Pulse exclusively through
the running API (`pulse.dashboard.api_client`) — never the database
directly (docs/architecture.md §2.1). `PULSE_API_URL` overrides the API base
URL if it isn't on `http://localhost:8000`.
