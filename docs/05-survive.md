# Stage: Survive — ✅ COMPLETE AND VALIDATED

**Goal:** from each duplicate group detected by One-Source Match (Step 4),
build **a single "survivor" record** by combining the best value of each
column between the master (`Match`) and its copy/copies (`Duplicate`).

## Real architecture (`JOB_05_Survive`)

```
onesource_match_matched.csv   ──┐
                                  ├→ Funnel_1 (Continuous funnel) → Survive_1 → Sequential_file_2 → customers_crm_survived.csv
onesource_match_duplicate.csv ──┘
```

- **Funnel_1**: type **"Continuous funnel"** (not "Sort Funnel" — the
  Survive stage itself handles sorting via its "Sort input data" option,
  so the Funnel just needs to concatenate the two streams, no extra
  work).
- **Survive_1**:
  - `Group identification column` = **`qsMatchSetID`** (the group key
    One-Source Match generated).
  - `Sort input data` = ✅ checked (required: the Funnel doesn't guarantee
    rows from the same `qsMatchSetID` stay contiguous).
  - Run in default mode (Parallel / Auto / Propagate) — no overrides,
    not needed with this data volume.
- **Sequential_file_2**: final output, `customers_crm_survived.csv`.

## Why `onesource_match_nonmatched.csv` was NOT merged into this job

`onesource_match_nonmatched.csv` has a **different schema** from
`matched`/`duplicate` (it doesn't include `qsMatchSetID`, only
`qsMatchDataID` and `qsMatchType`). That's why Survive only runs on the
two streams that share a schema; the 5 records with no duplicate (1005,
1008, 1009, 1012, 1013) get appended separately, in a final
Funnel/Append, to reach the 10-record master.

## Survive rule columns — real validated configuration

| # | Target column | Analyze column | Technique | Data |
|---|---|---|---|---|
| 1 | `AllColumns` | `qsMatchType` | **Equals** | `MP - Master record` |
| 2 | `PHONE_STD` | `PHONE_STD` | **At least one** | — |
| 3 | `ZIP_STD` | `ZIP_STD` | **Most frequent** | — |

### Key finding: use `qsMatchType` instead of trying "Minimum" on `CUST_ID`

The original plan proposed picking the master by "lowest CUST_ID" as a
tiebreaker. In practice, Survive's numeric comparison techniques
(`Equals`, `Not equals`, `Greater than`, `Less than`) **compare each
record against a fixed value typed into the `Data` column**, not against
each other within the group — there's no literal "Minimum"/"Maximum"
technique for picking the lowest value within a group.

The real, more robust solution: **One-Source Match already flags who the
master is** in the `qsMatchType` column (`"MP"` = master record on the
Match port, `"DA"` = duplicate on the Duplicate port). A single
`AllColumns` = `qsMatchType Equals "MP"` rule is enough for the master
record to donate all its columns to the survivor — no need to compare
text lengths or numeric values.

### Column-specific rules

- `PHONE_STD` → **At least one** (the "not blank / not null" equivalent):
  if one side of the pair has a populated phone and the other doesn't,
  the one that does wins.
- `ZIP_STD` → **Most frequent**: across the 5 seeded pairs the ZIP
  matches between master and copy, so any technique would give the same
  result; "Most frequent" is kept as the most robust choice in case of
  future divergence.

## UI bug observed (non-blocking)

When saving the stage, the side summary panel ("Survive rule columns," a
condensed view outside the edit modal) showed the value `"MP"` repeated
across all three rows of the `Data` column, including rows 2 and 3, which
correctly had an empty `Data` field inside the full edit modal. This
raised doubts, but **it was confirmed to be purely a cosmetic issue with
the summary panel**: the edit modal (`Define survive rule columns`)
always showed the correct values, and the real result of running the job
confirmed the rules were applied exactly as configured in the modal. No
additional fix was needed — just ignore the side summary and trust the
modal + the validation against the real output.

## Real (blocking) validation error when using numeric comparators

When trying the `Less than` technique on `CUST_ID` with an empty `Data`
column, saving failed with:

```
Rule 2: Numeric columns can only be compared to numeric data.
```

And when trying to leave a non-comparative technique (`At least one` /
`Most frequent`) with leftover text stuck in the `Data` field
(apparently a "ghost" value that persisted after typing and deleting),
saving kept repeating the error even after visually clearing the field:

```
Rule 2: This technique does not use data column. Please remove the data value.
```

**Solution that worked:** delete the buggy row (`⋮` → Delete) and create
a new row from scratch, never touching the `Data` field on techniques
that don't require it (`At least one`, `Most frequent`).

## Recurring problem #6: leading-zero loss in ZIP

The pattern already documented 4 times across the project (see
`08-troubleshooting.md`, point 15) showed up once more, this time in this
job's `Sequential_file_2` output: `ZIP_STD` got re-typed as numeric again,
losing the leading zero (`"03100"` → `"3100"`). Same fix as always:
manually re-type it to `VARCHAR` in `Sequential_file_2`'s output schema
and re-run the job.

## Final validated result

| Source port | Surviving CUST_ID | FULL_NAME | PHONE_STD | ZIP_STD | qsMatchSetID |
|---|---|---|---|---|---|
| Match (MP) | 1001 | Juan Perez Gomez | 5512345678 | 03100 | 1 |
| Match (MP) | 1003 | Maria Fernanda Lopez | 3398765432 | 44100 | 3 |
| Match (MP) | 1006 | Ana Torres | 2221112233 | 72000 | 6 |
| Match (MP) | 1010 | Jose Luis Hernandez | 4775556677 | 37000 | 10 |
| Match (MP) | 1014 | Sofia Castillo | 6647778899 | 22000 | 14 |

**5 records**, exactly one per `qsMatchSetID`, with the correct master
`CUST_ID` in every case and the ZIP's leading zero preserved after the
type fix.

Real file available at
`/outputs/survive_reports/customers_crm_survived.csv`.

## Pending to close out the deduplicated customer master — ✅ RESOLVED

These 5 survivors + the 5 records from `onesource_match_nonmatched.csv`
were merged in a standalone job, `JOB_06_MasterMerge`, to reach the
**10-record deduplicated customer master**. See full detail, architecture,
and findings in [`docs/05b-master-merge.md`](05b-master-merge.md).
