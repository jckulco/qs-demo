# Intermediate job: JOB_06_MasterMerge — ✅ COMPLETE AND VALIDATED

**Goal:** merge the 5 survivor records from `JOB_05_Survive` with the 5
records that never had a duplicate (`onesource_match_nonmatched.csv`) to
produce the **final deduplicated customer master**, the input for Step 6
(Two-Source Match).

It was built as a standalone job (not folded into `JOB_05_Survive`) to
keep each job focused on a single responsibility, and to be able to
re-run it without touching the already-validated Survive.

## Real architecture

```
customers_crm_survived.csv (5 rows, 105 cols)         ──┐
                                                          ├→ Funnel_1 (Continuous funnel) → Sequential file → customers_crm_clean.csv (10 rows, 99 cols)
onesource_match_nonmatched.csv (5 rows, 99 cols)      ──┘
```

- **Copy_1** (the `customers_crm_survived` branch): selects/drops the 6
  matching-diagnostic columns exclusive to this source (`qsMatchWeight`,
  `qsMatchPattern`, `qsMatchLRFlag`, `qsMatchExactFlag`,
  `qsMatchPassNumber`, `qsMatchSetID`) so its output schema matches
  `onesource_match_nonmatched.csv`'s (99 shared columns) exactly.
- **Funnel_1**: Continuous funnel, unions both branches now that schemas match.
- Output: `customers_crm_clean.csv`.

## Why the schema matched without needing manual NULLs

Comparing both sources column by column confirmed that
`onesource_match_nonmatched.csv` (99 columns) is an **exact subset** of
`customers_crm_survived.csv` (105 columns) — the difference is exactly
the 6 internal matching-diagnostic columns, which add no value to the
customer master. It was enough to deselect them on the `survived` branch
(via `Copy_1`) instead of trying to fill in missing columns on the
`nonmatched` side.

## Real problems found and solved

### 1. Reader schema misaligned after regenerating the source CSV

**Symptom:** first attempt at running the job, fatal error:
```
CDICO9999E: The data in row 0 for the qsMatchDataID column is invalid:
For input string: "1.10000000000000000E+01"
```

**Cause:** the reader stage `customers_crm_survivedcsv_1` had a cached
schema from an earlier version of the CSV (before the ZIP fix in Job 5).
When the file was regenerated, the column order/type ended up misaligned
— it was trying to read the `qsMatchWeight` value (scientific notation
format, e.g. `"1.1E+01"`) as if it were `qsMatchDataID` (a plain
integer).

**Solution:** force a schema re-detection from the current file in the
reader stage, instead of assuming the cached schema was still valid. Same
principle already documented in `08-troubleshooting.md`: *"when replacing
a source file with a corrected version, the schema in the stage that
reads it has to be refreshed manually."*

### 2. Implicit conversion warnings on new Funnel columns (non-blocking)

```
Funnel_1: Exterior_MXADDR: string → int32
Funnel_1: PHONE_STD: string → int64
```

Same recurring pattern in the project (a numeric-looking text column
getting re-typed). In this case **it didn't end up corrupting the real
data** — the output CSV kept the correct values as text — but the type
was preemptively fixed to `VARCHAR` in the schema before the Funnel, to
not rely on "it didn't get corrupted this time."

### 3. "Defaulting column in transfer" warnings — metadata noise, no real impact

**Symptom:** in a later run (after editing the schema for the fix in
point 2), 6 new warnings appeared:
```
Funnel_1: Defaulting "qsMatchWeight" in transfer from "inRec" to "outRec".
Funnel_1: Defaulting "qsMatchPattern" ...
Funnel_1: Defaulting "qsMatchLRFlag" ...
Funnel_1: Defaulting "qsMatchExactFlag" ...
Funnel_1: Defaulting "qsMatchPassNumber" ...
Funnel_1: Defaulting "qsMatchSetID" ...
```

**Investigation:** it was suspected that the column selection in `Copy_1`
had reverted while editing the schema for point 2 (re-exposing the 6
diagnostic columns). The real output CSV was validated: it was **still 99
columns**, with the 6 diagnostic columns still filtered out.

**Conclusion:** the warning happens during the *"checking operator"*
phase (before execution), when the engine detects that one branch's
declared metadata schema includes columns the other doesn't have, and
announces it will "default" (fill in) those columns to make the output
line up — but since the real column selection on the Funnel's output
already excludes them, they never make it into the final file. **It's
internal validation noise, not a functional problem.** Lesson: with any
new warning, always validate against the real output file before
assuming there's a problem — the third time in the project a log warning
turned out to be harmless (see also troubleshooting #17 on the cosmetic
Survive panel bug).

## Final validated result

**10 records, 99 columns** — exactly the expected master (5 survivors
with `qsMatchType = "MP"` + 5 without a duplicate with
`qsMatchType = "RA"`):

| CUST_ID | PHONE_STD | ZIP_STD | qsMatchType | Source |
|---|---|---|---|---|
| 1001 | 5512345678 | 03100 | MP | Survive |
| 1003 | 3398765432 | 44100 | MP | Survive |
| 1005 | 8122334455 | 64000 | RA | Nonmatched |
| 1006 | 2221112233 | 72000 | MP | Survive |
| 1008 | 5588990011 | 06700 | RA | Nonmatched |
| 1009 | NULL | 50000 | RA | Nonmatched |
| 1010 | 4775556677 | 37000 | MP | Survive |
| 1012 | NULL | 06140 | RA | Nonmatched |
| 1013 | 2223334455 | 72100 | RA | Nonmatched |
| 1014 | 6647778899 | 22000 | MP | Survive |

Real file at `/outputs/master_clean_reports/customers_crm_clean.csv`.

This dataset feeds **Step 6 (Two-Source Match)** as Source A, cross-checked
against `customers_erp_master.csv` (Source B).
