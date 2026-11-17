# Infrastructure setup

You run every command in this file yourself. Each command has a note saying what it
does. Run them from the repo root, in order.

Anything that can cost money is marked **💲 COST**.

---

## Stage 0: Local tools

| Tool | Check | Install if missing |
|---|---|---|
| Docker Desktop | `docker info` should not error | Open Docker Desktop. In **Settings → Resources**, set memory to **8 GB** (Airflow needs at least 4 GB). |
| uv (Python env manager) | `uv --version` | `brew install uv` |
| gcloud + bq CLIs | `gcloud --version` | already installed |

uv is a fast replacement for `pip` + `venv`. Each part of the repo (airflow, dbt,
mcp_server) gets its own isolated environment, so their dependencies can't clash.

---

## Stage 1: GCP setup

### 1. Load your settings into the shell

```bash
cp .env.example .env        # then edit .env and fill in the blanks
set -a; source .env; set +a # export every variable in .env into this shell
```

All the commands below use these variables (`$GCP_PROJECT_ID` and so on). If you open
a new terminal, run the `source` line again.

### 2. Make a separate gcloud configuration

```bash
gcloud config configurations create nsw-crash
gcloud config set account "$GCP_USER_EMAIL"
```

A *configuration* is a named set of gcloud settings (account, project, region). Your
existing default configuration stays pointed at your other project and is untouched.
To switch between them, run `gcloud config configurations activate default` or
`gcloud config configurations activate nsw-crash`.

### 3. Create the project and link billing (💲 COST)

```bash
gcloud projects create "$GCP_PROJECT_ID" --name="NSW crash risk"
gcloud config set project "$GCP_PROJECT_ID"
gcloud billing accounts list
gcloud billing projects link "$GCP_PROJECT_ID" --billing-account="$GCP_BILLING_ACCOUNT"
```

- `projects create`: a project is the container for every resource and every bill.
  Giving this work its own project means deleting it later removes everything.
- `billing accounts list`: shows your billing account ID. Put it in `.env` as
  `GCP_BILLING_ACCOUNT`, then run `source` again.
- `billing projects link`: **this is the step that makes charges possible.** We need
  it for two reasons. The BigQuery sandbox (no billing) blocks `MERGE`, `INSERT` and
  `DELETE`, and expires tables after 60 days. Cloud Run needs billing as well. Our
  usage is planned to stay inside the free tier.

Your account is a Google Workspace account, so your organisation's policies apply.
If `projects create` fails with a permission or organisation-policy error, paste me
the error.

### 4. Turn on the APIs

```bash
gcloud services enable \
  bigquery.googleapis.com \
  storage.googleapis.com \
  iam.googleapis.com \
  iamcredentials.googleapis.com \
  billingbudgets.googleapis.com \
  cloudresourcemanager.googleapis.com
```

Every GCP service starts switched off in a new project. Turning an API on is free.
`iamcredentials` is the one that lets you act as a service account (step 8).

Cloud Run, Artifact Registry and Cloud Build get turned on in Stage 6, when we need them.

### 5. Set a budget alert

```bash
gcloud billing budgets create \
  --billing-account="$GCP_BILLING_ACCOUNT" \
  --display-name="nsw-crash-risk guardrail" \
  --budget-amount=1 \
  --filter-projects="projects/$GCP_PROJECT_ID" \
  --threshold-rule=percent=0.5 \
  --threshold-rule=percent=1.0 \
  --threshold-rule=percent=1.0,basis=forecasted-spend
```

This emails you if this project's spend reaches 50% or 100% of 1 unit of your
billing currency, or is *forecast* to reach 100%.

**A budget is an alert, not a cap. It does not stop spending.** In an interview,
that's worth saying out loud.

### 6. Create the raw-file bucket

```bash
gcloud storage buckets create "gs://$GCS_BUCKET" \
  --location="$GCP_REGION" \
  --default-storage-class=STANDARD \
  --uniform-bucket-level-access \
  --public-access-prevention \
  --soft-delete-duration=0
```

- `--location=us-central1`: this flag defaults to the `US` *multi-region*, which is
  **not** in the free tier. Free Cloud Storage only applies in `us-central1`,
  `us-east1` and `us-west1`.
- `--uniform-bucket-level-access`: permissions are set once on the bucket, not on
  each file. That's simpler and easier to audit.
- `--public-access-prevention`: nothing in this bucket can ever be made public.
- `--soft-delete-duration=0`: by default, deleted or overwritten files are kept for
  7 days, **and you're billed for them**. Our DAGs overwrite files on purpose when
  they rerun, so we turn this off. In production you'd usually keep it on as an undo
  button.

### 7. Create the BigQuery datasets

```bash
bq --location="$GCP_REGION" mk --dataset \
  --description="Raw data loaded as-is from GCS" "$GCP_PROJECT_ID:$BQ_DATASET_RAW"
bq --location="$GCP_REGION" mk --dataset \
  --description="dbt staging and intermediate models" "$GCP_PROJECT_ID:$BQ_DATASET_STAGING"
bq --location="$GCP_REGION" mk --dataset \
  --description="dbt marts: tables that queries and the MCP server read" "$GCP_PROJECT_ID:$BQ_DATASET_MARTS"
```

A *dataset* is BigQuery's version of a schema: a folder of tables with its own
permissions and location.

The datasets live in `us-central1`, the **same region as the bucket**. Loading from a
bucket in a different location costs money for the transfer. Keeping everything in one
region keeps loads free and simple.

### 8. Create the pipeline service account and use it locally

```bash
gcloud iam service-accounts create sa-pipeline \
  --display-name="Airflow + dbt pipeline"

gcloud projects add-iam-policy-binding "$GCP_PROJECT_ID" \
  --member="serviceAccount:$PIPELINE_SA" --role="roles/bigquery.jobUser" --condition=None
gcloud projects add-iam-policy-binding "$GCP_PROJECT_ID" \
  --member="serviceAccount:$PIPELINE_SA" --role="roles/bigquery.dataEditor" --condition=None
gcloud storage buckets add-iam-policy-binding "gs://$GCS_BUCKET" \
  --member="serviceAccount:$PIPELINE_SA" --role="roles/storage.objectAdmin"

gcloud iam service-accounts add-iam-policy-binding "$PIPELINE_SA" \
  --member="user:$GCP_USER_EMAIL" --role="roles/iam.serviceAccountTokenCreator"
```

- A *service account* is a robot identity that code runs as, separate from you.
- `bigquery.jobUser` lets it run queries and load jobs.
- `bigquery.dataEditor` lets it create and replace tables.
- `storage.objectAdmin` is granted on **this bucket only**, not the whole project. It
  lets the account read and write files there.
- `serviceAccountTokenCreator` lets **you** borrow the account's identity. This is
  called *impersonation*.

Now create local credentials that act as the service account:

```bash
gcloud auth application-default login --impersonate-service-account="$PIPELINE_SA"
mkdir -p "$(dirname "$GOOGLE_APPLICATION_CREDENTIALS")"
cp ~/.config/gcloud/application_default_credentials.json "$GOOGLE_APPLICATION_CREDENTIALS"
chmod 600 "$GOOGLE_APPLICATION_CREDENTIALS"
```

**Why not download a JSON key?** A key file is a password that never expires. If it
leaks, anyone can act as the account until someone notices. With impersonation:

- No key is ever created.
- Access only works while your own login is valid.
- Audit logs show both you and the service account.

Many organisations block key creation entirely. Airflow's containers will read the
copied file through `GOOGLE_APPLICATION_CREDENTIALS`.

Two cautions:
- **This file is still a credential** (it holds your login's refresh token). It lives
  outside the repo, and `chmod 600` makes it readable only by you.
- **`application-default login` replaces your machine-wide default credentials.** If
  your other project relies on them, run plain `gcloud auth application-default login`
  again afterwards. Our copy in `~/.config/nsw-crash-risk/` is unaffected.

### 9. Verify

```bash
gcloud storage ls "gs://$GCS_BUCKET"
```
Lists the bucket. Empty output with no error means it exists and you can reach it.

```bash
bq ls --project_id="$GCP_PROJECT_ID"
```
Should list `raw`, `staging` and `marts`.

```bash
echo ok | gcloud storage cp - "gs://$GCS_BUCKET/_smoke/ok.txt" --impersonate-service-account="$PIPELINE_SA"
```
Writes a tiny file **as the service account**. This proves its permissions work.

```bash
uv run --no-project --with google-cloud-bigquery python -c "from google.cloud import bigquery; c=bigquery.Client(project='$GCP_PROJECT_ID'); print(list(c.query('select session_user() as who').result())[0].who)"
```
Runs a query from Python using the copied credentials. It should print the
`sa-pipeline@…` address, not your own email.

```bash
gcloud storage rm "gs://$GCS_BUCKET/_smoke/ok.txt"
```
Cleans up the test file.

---

## Teardown (when you're finished with the project)

```bash
gcloud projects delete "$GCP_PROJECT_ID"
```

Shuts down the project and everything in it. Billing stops, and the project can be
restored for 30 days. **This is destructive.** Only run it when you mean it.
