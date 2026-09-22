# Step-by-step guide: end-to-end QualityStage demo

> **Current status: PROJECT COMPLETE.** All **6 steps** of the demo
> (Investigate, Standardize, Match Frequency, One-Source Match, Survive,
> Two-Source Match) are complete and validated end to end on a real
> instance, along with the 3 prep/consolidation jobs
> (`JOB_06_MasterMerge`, `JOB_07_StandardizeERP`,
> `JOB_08_MatchFrequency_TwoSource`). The final result
> (`JOB_09_TwoSourceMatch`) answers the original business question: 6
> customers exist in both systems, 2 are ERP-exclusive, 4 are
> CRM-exclusive. **30 real problems** were solved along the way,
> documented in [`docs/07-troubleshooting.md`](07-troubleshooting.md).
>

**Business case:** the Marketing team has a CSV exported from the CRM
(`customers_crm.csv`) with potentially duplicated and poorly-typed
customers (inconsistent capitalization, different street abbreviations,
"Ma." vs "Maria", abbreviated and unabbreviated states, phone numbers with
and without dashes). Before loading it into the Data Warehouse, we need
to:

1. Know **how dirty** the data actually is (Investigate).
2. **Normalize it** into a consistent format (Standardize).
3. **Detect internal duplicates** within the CRM itself (Match Frequency +
   One-Source Match).
4. Pick, from each duplicate group, **the most complete record** (Survive).
5. **Cross-check the deduplicated master against the ERP** to know which
   customers already exist there and which are new (Two-Source Match).

---

## Step 0 — Set up the project

1. In Cloud Pak for Data / watsonx.data integration, create a new
   **project**, e.g. `qs-demo-clientes`.
2. Upload `data/customers_crm.csv` and `data/customers_erp_master.csv` as
   **data assets** in the project.
3. Create a new DataStage flow: `JOB_01_Investigate_Standardize_Match`.
4. Drag a **File Connector / Local file** stage pointing at
   `customers_crm.csv` as the source.

---

## Step 1 — Investigate

📄 Full detail: [`docs/01-investigate.md`](01-investigate.md) · Spec: [`specs/investigate_spec.md`](../specs/investigate_spec.md)

Drag the **Investigate** stage from the *Data Quality* group in the
palette. Connect it to the CRM source output (via a `Copy` that fans out
into several branches — one per column group).

- Mode: **Word Investigate** (not Character — it had a UI bug on the
  tested instance, see `08-troubleshooting.md`) over 4 branches:
  `FULL_NAME` (rule `MXNAME`), `ADDRESS_LINE` (rule `MXADDR`),
  `CITY`+`STATE`+`ZIP` together (rule `MXAREA`), and `PHONE` (no rule
  set, via a separate Transformer).
- Goal: generate a **word pattern report** (frequency report) showing,
  for example, how many records use "Av." vs "Avenida," what name
  variants exist, and how inconsistent the ZIP is.
- Run the job and check the frequency report: this is where you
  **confirm** the dirty patterns we already seeded into the CSV
  (duplicates from text variation, blank fields, different phone
  formats).

This step doesn't modify the data — it only gives you evidence to confirm
the rules for the next stage.

**✅ Validated:** real Investigate was run on `ADDRESS_LINE` (Word
investigation + MXADDR rule), `FULL_NAME` (MXNAME rule), `CITY`+`STATE`+`ZIP`
together (MXAREA rule), and `PHONE` (via Transformer, no rule set). All 4
real frequency reports are in `/outputs/investigate_reports/`.

---

## Step 2 — Standardize

📄 Full detail: [`docs/02-standardize.md`](02-standardize.md) · Spec: [`specs/standardize_rules.md`](../specs/standardize_rules.md)

**✅ Complete and validated.** The real Standardize ended up split across
2 jobs:

- In `JOB_01`, two Transformers generate `PHONE_STD.csv` and
  `ZIP_STD.csv` (cleanup with no rule set, since `MXPHONE`/`MXZIP` don't
  exist).
- In `JOB_02_Standardize`: 3 **Standardize** stages (`MXNAME` on
  `FULL_NAME`, `MXADDR` on `ADDRESS_LINE`, `MXAREA` on
  `CITY`+`STATE`+`ZIP` together) + the 2 external sources above, joined
  with **4 chained Joins** on `CUST_ID`.

Output: `customers_crm_standardized.csv` (97 columns, 15 rows), available
in `/outputs/standardize_reports/`.

6 real problems were solved in this step (contamination from a
misconfigured literal, expression syntax, data-type propagation across 3
different layers, unreliable preview views) — all in
[`docs/08-troubleshooting.md`](08-troubleshooting.md), points 5 to 10.

---

## Step 3 — Match Frequency

📄 Detail: [`docs/03-match-frequency.md`](03-match-frequency.md)

**✅ Complete.** Match Frequency stage over
`customers_crm_standardized.csv`, using the `Match*` columns
(`MatchPrimerNombre_MXNAME`, `MatchApellido_MXNAME`,
`MatchNombreCalle_MXADDR`, `PHONE_STD`, `ZIP_STD`) — not the display
columns. Output: `customers_crm_match_frequency.csv`, a specialized
internal structure (not customer data) that feeds Step 4.

---

## Step 4 — One-Source Match (internal CRM deduplication)

📄 Detail: [`docs/04-one-source-match.md`](04-one-source-match.md) · Spec: [`specs/match_specification_onesource.md`](../specs/match_specification_onesource.md)

**✅ Complete and validated.** The Match Specification is built as a
standalone project asset (not inside the job), needs to be published
("Provision") before it's visible from the job, and the stage needs the
**"Duplicate"** port checked — without it, 33% of the records get
silently dropped. Real result: the 5 duplicate pairs seeded in the
dataset (1001/1002, 1003/1004, 1006/1007, 1010/1011, 1014/1015) were
detected exactly, 15 of 15 records accounted for.

---

## Step 5 — Survive

📄 Detail: [`docs/05-survive.md`](05-survive.md)

**✅ Complete and validated.** Real job (`JOB_05_Survive`): `Funnel_1`
(Continuous funnel, unions `onesource_match_matched.csv` +
`onesource_match_duplicate.csv`) → `Survive_1` (grouped by
`qsMatchSetID`, "Sort input data" enabled) → `Sequential_file_2` →
`customers_crm_survived.csv`.

Real rules that worked:
- `AllColumns` = `qsMatchType Equals "MP - Master record"` — takes
  advantage of the fact that One-Source Match already flags the master
  record, instead of trying to compare `CUST_ID` numerically (the
  `Less than`/`Greater than` techniques compare against a fixed value,
  not between records in the group).
- `PHONE_STD` = `At least one` (the "not blank" equivalent).
- `ZIP_STD` = `Most frequent`.

Result: **5 records**, one per `qsMatchSetID`, with the correct master
`CUST_ID`s (1001, 1003, 1006, 1010, 1014). Real output at
`/outputs/survive_reports/customers_crm_survived.csv`.

**Pending:** merge these 5 survivors with the 5 records from
`onesource_match_nonmatched.csv` (different schema, no `qsMatchSetID`) in
a final Funnel/Append to reach the 10-record master that feeds Step 6.

---

## Intermediate job — Merge Survive + Nonmatched (`JOB_06_MasterMerge`)

📄 Detail: [`docs/05b-master-merge.md`](05b-master-merge.md)

**✅ Complete and validated.** Standalone job (kept separate from
`JOB_05` to keep responsibilities isolated): `customers_crm_survived.csv`
(with its 6 matching-diagnostic columns dropped via `Copy_1`) +
`onesource_match_nonmatched.csv` → `Funnel_1` (Continuous funnel) →
`customers_crm_clean.csv`.

Result: **10 records, 99 columns** — the final deduplicated customer
master (5 with `qsMatchType = "MP"` from Survive, 5 with
`qsMatchType = "RA"` from Nonmatched). Real output at
`/outputs/master_clean_reports/customers_crm_clean.csv`.

3 more problems were solved in this job (misaligned reader schema after
regenerating the source CSV, implicit-conversion warnings on new columns,
"defaulting column" warnings that turned out to be harmless metadata
noise) — documented in `08-troubleshooting.md`, points 19 to 21.

---

## Prep job — Standardize the ERP (`JOB_07_StandardizeERP`)

📄 Detail: [`docs/06a-standardize-erp.md`](06a-standardize-erp.md)

**✅ Complete and validated.** Same pattern as `JOB_02_Standardize`,
applied to `customers_erp_master.csv`: 3 Standardize branches
(`NAME`/MXNAME, `STREET`/MXADDR,
`CITY+STATE_CODE+POSTAL_CODE`/MXAREA) + 2 Transformers (`ZIP_STD`,
`TELEPHONE_STD`) joined by 4 chained Joins on `ERP_ID`.

Result: **8 records, 98 columns** — `customers_erp_standardized.csv`,
comparable 1:1 with the CRM via `ZIP_STD` ↔ `ZIP_STD` and
`TELEPHONE_STD` ↔ `PHONE_STD`. Real output at
`/outputs/erp_standardize_reports/customers_erp_standardized.csv`.

3 new problems were solved in this job — the most notable being a new
finding for the project: a Transformer output column named `_STD` can be
a **disguised pass-through** (a derivation like `link.column` with no
function wrapping it at all), which only shows up by checking real edge
cases in the data, not the records that already looked "clean." Documented
in `08-troubleshooting.md`, points 22 to 24.

---

## Prep job — Match Frequency, per source (`JOB_08_MatchFrequency_TwoSource`)

📄 Detail: [`docs/06b-match-frequency-twosource.md`](06b-match-frequency-twosource.md)

**✅ Complete and validated.** The first design combined CRM + ERP into a
single stream with a `Funnel` before computing one shared frequency
table — that approach doesn't work here (`Funnel` doesn't union
mismatched schemas; see the finding in the doc above) and was abandoned.
The final design runs **two fully independent branches**:

- `customers_crm_clean.csv` → Match Frequency → `customers_crm_match_frequency.csv` (142 records)
- `customers_erp_standardized.csv` → Transformer (renames 3 columns with an `_ERP` suffix) → Match Frequency → `customers_erp_match_frequency.csv` (121 records)

Each file carries exactly the column names its side uses in the
Two-Source Match Specification — nothing shared, nothing over-renamed.
Real outputs at `/outputs/match_frequency_twosource_reports/`.

---

## Step 6 — Two-Source Match (cross-check against the ERP) — ✅ COMPLETE AND VALIDATED

📄 Real execution detail: [`docs/06c-two-source-match-execution.md`](06c-two-source-match-execution.md)
· Original theoretical plan (historical reference):
[`docs/06-two-source-match.md`](06-two-source-match.md) ·
Real spec: `twosource_match_crm_erp` (a published asset in the project,
not a file — see detail in the execution doc)

Real job: `JOB_09_TwoSourceMatch`, using as inputs
`customers_erp_standardized.csv` (`JOB_07`) and the two independent
frequency files from `JOB_08` (see
[`docs/06b-match-frequency-twosource.md`](06b-match-frequency-twosource.md)).

This was the most complex step in the entire project — 6 distinct
findings documented in `08-troubleshooting.md` (points 25-30), including
the fact that the stage needs 4 input links (not 2), that `Funnel`
doesn't union mismatched schemas, and that the theoretical plan's cutoffs
didn't hold up against the real weights computed from the real data.

**Final result:**
- **Match (6)**: Juan Perez Gomez, Maria Fernanda Lopez, Carlos Alberto
  Sanchez, Ana Torres, Roberto Diaz Martinez, Jose Luis Hernandez.
- **Data nonmatched (2)**: Andrea Mendoza, Diego Ramirez (ERP only).
- **Reference nonmatched (4)**: Laura Gonzalez, Patricia Ramirez, Fernando
  Ortiz, Sofia Castillo (CRM only).
- **Clerical (0)**.

Real outputs in `/outputs/twosource_match_reports/`.

---

## Step 7 — Validate results

Compare your outputs against `data/expected_output_notes.md` (included in
this repo) to confirm the duplicate groups and the ERP cross-checks match
what's expected.

## Step 8 — watsonx Data Integration

- The real project export (the 9 jobs, ready to import) is already
  included in `/platform-export` — see `platform-export/README.md` for
  the detail on what's in it and how to import it into your own
  instance.