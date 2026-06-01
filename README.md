# BenchEngine

A self-hosted platform for collecting, analysing, and comparing PC hardware
performance telemetry. Contributors upload raw capture logs from tools like
**HWInfo** and **CapFrameX**; Engine Benchmark (EB) cleans them, extracts windowed
performance features, runs a battery of statistical and clustering analyses, and
serves the results through an interactive dashboard and a component leaderboard.

It's designed to run comfortably on a **Raspberry Pi 4** behind a **Cloudflare
Tunnel**. No open router ports, no exposed home IP and has rate limiting, upload
validation, security headers, and anonymous contributor handling built in.

---

## What it does

- **Ingestion** — Streams uploaded CSVs to disk with a size cap, validates them,
  normalises inconsistent HWInfo/CapFrameX column names, builds a clean
  timestamp, and hardens noisy sensor data.
- **Feature extraction** — Slices each run into windows and computes FPS
  statistics (mean, std, 1% / 0.1% lows), frametime percentiles (p99 / p99.9),
  and thermal averages, persisted to **DuckDB**.
- **Analysis engine** — Bootstrap confidence intervals, distribution and tail
  analysis, throttling and stability detection, hypothesis testing,
  reproducibility scoring, plus composite / relative / benchmark scoring.
- **Workload clustering** — Groups runs into performance regimes with K-means and
  reconciles free-text game names to canonical workloads via fuzzy matching.
- **Component leaderboards** — Ranks CPUs / GPUs / RAM by normalised performance
  across workloads.
- **Dashboards** — A public Plotly dashboard (`/`), a password-protected admin
  monitor (`/admin`), and an optional standalone Streamlit view.

## Architecture

```
                 upload (CSV)
                      │
        ┌─────────────▼──────────────┐
        │  FastAPI app (app/)         │   ← rate limiting, upload validation,
        │  middleware + JSON API      │     security headers, monitoring
        └─────────────┬──────────────┘
                      │
        ┌─────────────▼──────────────┐
        │  Ingestion pipeline         │   load → clean → harden → timestamp
        │  (engine/ingestion/)        │
        └─────────────┬──────────────┘
                      │
        ┌─────────────▼──────────────┐
        │  Feature builder            │   windowed FPS / frametime / thermals
        │  (engine/features/)         │
        └─────────────┬──────────────┘
                      │
            ┌─────────▼─────────┐
            │   DuckDB           │   raw_metrics · features · systems · runs
            │   (data/)          │   + component-platform tables
            └─────────┬─────────┘
                      │
        ┌─────────────▼──────────────┐
        │  Analysis + scoring engine  │   stats · scoring · clustering
        │  (engine/analysis/)         │
        └─────────────┬──────────────┘
                      │
        ┌─────────────▼──────────────┐
        │  DataService (app/services) │   → JSON API → dashboards
        └─────────────────────────────┘
```

## Tech stack

| Layer        | Tools                                         |
|--------------|-----------------------------------------------|
| Web / API    | FastAPI, Uvicorn                              |
| Data         | Polars, DuckDB                                |
| Stats / ML   | NumPy, SciPy, scikit-learn                    |
| Frontend     | Plotly dashboards, Streamlit (optional)       |
| Deployment   | systemd, Cloudflare Tunnel (Raspberry Pi)     |

---

## Quickstart (local development)

> Requires **Python 3.11+**.

```bash
git clone https://github.com/<owner>/<repo>.git
cd <repo>

python -m venv .venv
source .venv/bin/activate

pip install -r requirements.txt
```

Initialise the database and (optionally) ingest any sample CSVs you place in
`data/raw/`:

```bash
python run_pipeline.py
```

Run the API + dashboard:

```bash
uvicorn app.main:app --reload --port 8000
```

Then open:

- **http://127.0.0.1:8000/** — public dashboard
- **http://127.0.0.1:8000/admin** — admin monitor (returns 503 until you set an
  admin password; see below)

### Configuration

All tunables are environment variables with conservative defaults. Copy the
example file and edit it:

```bash
cp deploy/benchengine.env.example .env
chmod 600 .env
# set PF_ADMIN_PASSWORD to something long and random to enable /admin
```

Key variables (full list in `deploy/benchengine.env.example` and
`app/security/settings.py`):

| Variable                 | Default | Purpose                                  |
|--------------------------|---------|------------------------------------------|
| `PF_ADMIN_USER`          | `admin` | Admin dashboard username                 |
| `PF_ADMIN_PASSWORD`      | _unset_ | Admin password — **`/admin` is disabled until set** |
| `PF_RATE_GENERAL_PER_MIN`| `90`    | Sustained general requests / IP / min    |
| `PF_RATE_UPLOAD_PER_MIN` | `5`     | Sustained uploads / IP / min             |
| `PF_MAX_UPLOAD_MB`       | `64`    | Max accepted upload size                 |

### Optional: Streamlit dashboard

```bash
pip install "streamlit~=1.40"
streamlit run dashboard/app.py
```

---

## Deployment

Putting the site online (Raspberry Pi setup, the systemd service, secrets, and
the Cloudflare Tunnel) is documented step by step in
**[`deploy/SECURITY_SETUP.md`](deploy/SECURITY_SETUP.md)**. The in-app hardening
listed there runs automatically — the guide covers only the parts that require
host-level setup.

---

## Project layout

```
.
├── app/                      FastAPI application
│   ├── main.py               app entry point, middleware, JSON API routes
│   ├── routers/              admin + component-platform routers
│   ├── security/             rate limiting, upload guard, headers, settings
│   ├── services/             DataService — query + analysis orchestration
│   └── static/               dashboard.html, admin.html
├── engine/                   ingestion + analysis engine (framework-agnostic)
│   ├── ingestion/            load → clean → harden → timestamp → pipeline
│   ├── features/             windowed feature builder
│   ├── analysis/             stats, scoring, clustering, leaderboards
│   ├── components/           component analyzer/registry, workload normaliser,
│   │                         anonymous contributor hashing
│   ├── db/                   DuckDB schema + connection management
│   └── monitoring/           server monitor (request/upload/exception stats)
├── scripts/                  CLI entry points (ingest, build_features, reports)
├── deploy/                   systemd unit, Cloudflare config, env example, guide
├── dashboard/                optional standalone Streamlit app
├── run_pipeline.py           one-shot: init db → ingest → build features
├── check_db.py               quick sanity check of stored timeseries
├── requirements.txt
├── requirements-dev.txt
└── pyproject.toml            tooling config (ruff, pytest)
```

See **[`docs/PROJECT_STRUCTURE.md`](docs/PROJECT_STRUCTURE.md)** for a
file-by-file map.

---

## Development

```bash
pip install -r requirements.txt -r requirements-dev.txt

ruff check .          # lint
ruff format .         # format
pytest                # run tests (add yours under tests/)
```

CI runs lint + tests on every push and pull request — see
`.github/workflows/ci.yml`.

## Contributing

Contributions are welcome. Please read [`CONTRIBUTING.md`](CONTRIBUTING.md)
first. To report a security issue, see [`SECURITY.md`](SECURITY.md).

## License

Released under the [MIT License](LICENSE).
