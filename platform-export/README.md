# Platform export — the real Cloud Pak for Data project

This folder is the actual project export from **IBM Cloud Pak for Data /
watsonx.data integration** (`Manage project > Export project`), taken at
the point where every job in this demo was already built, run, and
validated. It's not a mockup or a recreation — it's the project itself.

## What's inside

- **9 real DataStage flows** (`data_intg_flow`), matching the 9 jobs
  documented in `/docs`:
  - `JOB_01_Investigate_Standardize_Match`
  - `JOB_02_Standardize`
  - `JOB_03_MatchFrequency`
  - `JOB_04_OneSourceMatch`
  - `JOB_05_Survive`
  - `JOB_06_MasterMerge`
  - `JOB_07_StandardizeERP`
  - `JOB_08_MatchFrequency_TwoSource`
  - `JOB_09_TwoSourceMatch`
- **2 Data Definitions**, created manually to work around the schema-type
  sniffing issue documented in `docs/08-troubleshooting.md` (point 28) —
  `customers_erp_standardized` and `customers_crm_clean`, both with the 5
  match columns explicitly typed as `VARCHAR`.
- **Data asset connections**, pointing to the CRM and ERP CSV files and
  every intermediate output generated along the way.
- **Two test flows** (prefixed `.test_`) — these are the two published
  Match Specifications (`onesource_match_crm` and `twosource_match_crm_erp`)
  as they get represented in an export. Their real configuration (blocking
  columns, match columns, m/u-probabilities, cutoffs) is documented in
  human-readable form in `docs/04-one-source-match.md` and
  `docs/06c-two-source-match-execution.md`.

## How to import it

1. In your own Cloud Pak for Data / watsonx.data integration instance, go
   to **Manage project → Import project**.
2. Point it at this `platform-export` folder (or zip it first if your
   instance requires a `.zip` upload — check your platform's import
   dialog).
3. Once imported, you'll have all 9 jobs on your canvas, wired up and
   ready to run — but you'll need to re-upload `data/customers_crm.csv`
   and `data/customers_erp_master.csv` as data assets in your project
   first, since the export references them by connection, not by
   embedding the file contents.
4. Re-publish both Match Specifications (`Provision` tab on each asset) —
   published state doesn't always survive an export/import round-trip.

## Why this is here

Everything in `/docs` describes *how* to build this from scratch, step by
step, including every dead end. This folder is the fast path: if you'd
rather see the finished thing first and reverse-engineer it, or just
confirm your own build matches the real one, import this and compare.
