# BIFlow

**From raw CSV files to a business dashboard you can trust, in one command.**

BIFlow is a business-intelligence pipeline built from small, independent services. You give it raw data
(for example an e-commerce export), and it ingests it, checks its quality, cleans it, models it, computes KPIs,
looks for trends and anomalies, and gives you a dashboard that explains what changed and why.

The main idea: **code computes every number.** A language model is optional. When it is turned on, it only writes
the text around the numbers, and every number it writes is checked against the real results before it is shown.
Every step is also logged, so you can trace any KPI back to the raw rows it came from.

---

## Quick start

You need **[Docker Desktop](https://www.docker.com/products/docker-desktop/)** (running) and **[Git](https://git-scm.com/downloads)**.
You don't need Python, a database or any data download: demo data is included.

```bash
git clone https://github.com/khairovic/BI-Flow.git
cd BI-Flow
```

Start everything:

| Windows (PowerShell) | macOS / Linux |
|---|---|
| `powershell -ExecutionPolicy Bypass -File start.ps1` | `./start.sh` |

The script checks Docker, creates your `.env` file, builds the services, runs the pipeline on a small sample of the
[Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) e-commerce dataset, and opens the dashboard at
**http://localhost:8501**.

- The first start takes **5–15 minutes** (mostly downloading Docker images). After that it starts in seconds.
- The script asks for an **optional** LLM API key. Press Enter to skip: everything still works, using built-in text
  templates. With a key (a free [Groq](https://console.groq.com/keys) key works, or any OpenAI-compatible provider),
  the summaries and explanations are written by the model.

### Other options

| Command (Windows: `.\start.ps1 ...` · macOS/Linux: `./start.sh ...`) | What it does |
|---|---|
| `-Dataset bike_share` / `bike_share` | Runs a second, unrelated sector (city bike sharing), set up only with config files |
| `-Full` / `--full` | Downloads and runs the **full** Olist dataset (100k orders, about 4 minutes) |
| `-Stop` / `--stop` | Stops everything |

If the Kaggle download is blocked on your network, download the dataset by hand, unzip the 9 CSV files into
`data/raw/olist/`, and run `start.ps1 -Dataset olist` / `./start.sh olist`.

---

## What you get in the dashboard

| Tab | What you see |
|---|---|
| **Overview** | A summary of the run, one tile per KPI, and the status of every pipeline step |
| **KPI explorer** | Trends and breakdowns, plus **"why did it move?"**: a change split by segment |
| **Insights and actions** | Trends, anomalies (such as the Black Friday spike) and recommendations |
| **Data quality** | A 0–100 quality score before and after cleaning, and what changed |
| **Lineage** | A graph from the raw tables to each KPI, plus an audit report |
| **Explain** | Ask a question in plain words ("why did sales drop?") and get a fact-checked answer with sources |

You can also upload your own files from the sidebar, as long as they have the same structure as an existing dataset.
A tiny test set is in [`tests/fixtures/upload_demo/`](tests/fixtures/upload_demo/).

---

## How it works

Each run goes through these steps:

```
ingest → check raw data → profile → score → clean → profile again → validate → score again
       → model (dbt on DuckDB) → compute KPIs → analyse → build dashboard → lineage → audit → summary
```

A **gateway** service (built with LangGraph) runs the steps in order and is the only service that calls the others.
If a step fails, the run still goes to the audit step, so the failure gets explained. Each run saves the exact
config it used, and you can resume a run or redo it from any step.

| Service | Port | Role |
|---|---|---|
| `gateway` | 8000 | Runs the pipeline, handles resume, writes the run summary |
| `etl` | 8101 | Builds the data models with dbt on DuckDB, with tests and reconciliation |
| `auditor` | 8102 | Lineage graph, audit report, "explain this KPI" with sources, fact-checking |
| `profile_clean` | 8103 | Profiles the data and cleans it (rules, ML, optional LLM advice), with a review queue |
| `ingest_quality` | 8104 | Reads CSV / SQL / API sources, checks schemas, computes the quality score |
| `semantic_kpi` | 8105 | KPI catalog, KPI search, turns a question into the right KPI |
| `analyst` | 8106 | Trends, anomalies, driver breakdowns, recommendations, written summaries |
| `dashboard` | 8501 | The Streamlit web app |

**Adding a new dataset doesn't need any code**, only config files: `configs/datasets/<name>.yaml`,
`configs/marts/<domain>/`, `configs/kpis/<domain>.yaml` and `configs/recommendations/<domain>.yaml`.
The `bike_share` dataset is a complete example.

**Tech stack:** Python 3.11, FastAPI, LangGraph, dbt + DuckDB, pandas, pandera, scikit-learn, statsmodels,
ChromaDB, Plotly, Streamlit, Docker Compose.

---

## Command line

Everything in the dashboard can also be run from the terminal:

```bash
docker compose exec gateway python -m app.cli <command>
```

| Command | What it does |
|---|---|
| `run --dataset olist_sample` | Runs the pipeline (`olist_sample`, `bike_share` or `olist`) |
| `run --resume RUN_ID --from STAGE` | Continues a run, or redoes one step and everything after it |
| `list` · `status RUN_ID` · `graph` | Lists runs, shows one run, prints the pipeline graph |
| `bench --dataset olist_sample --runs 5` | Measures time-to-insight |
| `eval --all` | Runs every service's self-evaluation |

---

## Project structure

```
services/<name>/   one folder per service: app/, tests/, eval/, mock/, Dockerfile
contracts/         shared request/response models used by all services
configs/           dataset, model, KPI and recommendation configs
config/models.yaml LLM model names (API, Ollama, low-RAM)
prompts/           versioned prompts
data/demo/         demo data included in the repo
tests/fixtures/    1% Olist sample used for the demo and the tests
scripts/           start helpers and dev tools
docs/              technical spec, results, future work
```

## Development

- Code is mounted into the containers, so after a change `docker compose restart <service>` is enough.
- Run one service's tests:
  `docker compose run --rm --no-deps -e BIFLOW_MOCK=1 -e FAKE_LLM=1 <service> sh -c "pip install -q pytest && python -m pytest -q"`
- Logs of one service: `docker compose logs -f <service>`

## Troubleshooting

- **The build fails with a network error:** run the start script again. Downloads resume from the cache.
- **"Docker is not running":** open Docker Desktop and wait until it says it is running.
- **A port is already in use** (8000, 8101–8106, 8501): stop the program using it, or stop an older BIFlow with `-Stop`.
- **LLM calls fail even though the key is right:** if `docker info` shows an HTTP proxy, uncomment the two proxy
  lines in `.env` and start again.

## Documentation

- [Technical spec](docs/BIFlow_Technical_Spec.md)
- [Benchmarks and evaluations](docs/results/)
- [Future work](docs/FUTURE_WORK.md)
