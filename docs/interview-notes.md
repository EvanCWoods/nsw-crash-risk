# Interview notes

One section per stage: what we built, why, and the questions it prepares you for.
Practise answering them out loud.

---

## Stage 1: GCP setup

**What we built:** a dedicated Google Cloud project with:
- a Cloud Storage bucket for raw files
- three BigQuery datasets (`raw`, `staging`, `marts`)
- a pipeline service account
- a budget alert

Everything lives in one free-tier region (`us-central1`).

**Why it's set up this way:**
- **A separate project** keeps billing and permissions apart, and makes cleanup a
  single command.
- **One region** keeps loads from Cloud Storage into BigQuery free. Moving data
  between locations costs money.
- **Impersonation instead of key files.** Code runs as a service account, and you
  borrow its identity with your own login, so no long-lived key file can leak.
- **Soft delete turned off** on the raw bucket. The pipeline overwrites files on
  purpose when it reruns, and soft-deleted copies are billed.

**Interview questions**

1. *How do you give a pipeline access to GCP without putting secrets in the repo?*

   Use a service account that has only the access it needs.
   - Locally, impersonate it instead of downloading a key.
   - On Cloud Run, attach it to the service.

   Credentials stay out of git, and `.env.example` documents which settings are needed.

2. *How do you control cloud costs on a project like this?*
   - Use free-tier regions, and keep storage and compute in the same place.
   - Load in batches rather than streaming.
   - Turn off soft delete for raw files you overwrite on purpose.
   - Set a budget alert, knowing it doesn't cap spend.
   - Later, cap the bytes each query can scan.

3. *When would you choose a single region over a multi-region?*
   - A single region is cheaper and avoids cross-location transfer costs.
   - A multi-region gives more redundancy.
   - Data residency can decide it: Australian client data may have to stay in
     `australia-southeast1`.
