# NSW crash risk pipeline

> **Status: in progress.** This README is the project guide while we build. It becomes
> the final project README in the last stage.

## What we're building, in one paragraph

A small data platform that answers the question **"how does weather affect road crashes
in NSW?"** It downloads five years of NSW government crash records (2021–2025) and daily
weather for the same period. It stores them in Google's data warehouse (BigQuery), and
cleans and combines them into tidy tables. Then it lets an AI assistant such as Claude
answer questions about the data in plain English, for example: *"Were crashes in the
Hunter region more severe on rainy days in 2024?"*

It's a portfolio project for a data engineering interview. Every piece is something
data engineers use daily, and you should be able to explain each one.

## How the data flows

```
 NSW crash files (xlsx)      Open-Meteo weather API
          │                          │
          └──────────┬───────────────┘
                     ▼
     ① AIRFLOW (on your laptop, in Docker)
        downloads the data on a schedule
                     │
                     ▼
     ② CLOUD STORAGE  (a bucket of raw files, kept unchanged)
                     │
                     ▼
     ③ BIGQUERY "raw" dataset  (the files loaded as tables)
                     │
                     ▼
     ④ DBT  (SQL files that clean, join and test the data)
        raw → staging → marts
                     │
                     ▼
     ⑤ BIGQUERY "marts" dataset  (final, question-ready tables)
                     │
                     ▼
     ⑥ MCP SERVER on Cloud Run  (lets AI assistants query the marts)
```

## The stages, and where we are

| # | Stage | What you end up with | Status |
|---|---|---|---|
| 0 | Local tools | Docker, uv and gcloud working on your Mac | ⏳ **you are here** |
| 1 | GCP setup | A Google Cloud project with a bucket, 3 BigQuery datasets and a service account | ⏳ **you are here** |
| 2 | Ingest | Airflow running locally, loading crash + weather data into BigQuery | not started |
| 3 | dbt staging | Cleaned, typed, tested versions of the raw tables | not started |
| 4 | dbt marts | Final tables joining crashes to weather | not started |
| 5 | MCP server | A Python server that answers questions from the marts | not started |
| 6 | Deploy | The MCP server running on Cloud Run | not started |
| 7 | README | Architecture diagram, setup guide, honest production notes | not started |

After each stage you get a plain-language explanation and the interview questions
it prepares you for. These are collected in [docs/interview-notes.md](docs/interview-notes.md).

## How work is tracked

Every piece of work is an issue in the Linear project
[evan-personal-data](https://linear.app/proudlyhumantm/project/evan-personal-data-1b8116258582).
Each milestone is one stage above. One issue gets one branch and one pull request,
and merging the PR closes the issue. The full workflow is in
[docs/development-process.md](docs/development-process.md).

## What to do right now

1. **EVA-36:** install uv (`brew install uv`), then open **Docker Desktop** and set
   memory to 8 GB (Settings → Resources).
2. **EVA-37:** follow **[infra/README.md](infra/README.md)** from top to bottom. It
   creates the Google Cloud project, one command at a time, with a note on what each
   one does.
3. When the last check in step 9 prints `sa-pipeline@…`, Stage 1 is done.
   Tell Claude, and Stage 2 (EVA-38) starts.

## Repo layout

| Folder | What goes in it | Filled in at |
|---|---|---|
| `infra/` | The gcloud/bq setup commands you run | Stage 1 ✅ |
| `airflow/` | The pipelines (DAGs) that download and load data | Stage 2 |
| `dbt/` | The SQL models and tests | Stages 3–4 |
| `mcp_server/` | The AI-assistant query server | Stages 5–6 |
| `docs/` | Architecture notes | Stage 7 |

`.env.example` lists every setting the project needs. Copy it to `.env` (which git
ignores) and fill it in. Secrets never go in the repo.

## Data sources and licences

- **NSW Road Crash Data 2021–2025**, Transport for NSW Open Data, CC BY.
- **Historical weather**, [Open-Meteo](https://open-meteo.com), CC BY 4.0 (free for
  non-commercial use).
