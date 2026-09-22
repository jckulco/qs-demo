# Job: JOB_07_StandardizeERP — ✅ COMPLETE AND VALIDATED

**Goal:** apply to `customers_erp_master.csv` the same Standardize
treatment already validated on the CRM (`JOB_02_Standardize`), so both
sources end up with comparable columns before Two-Source Match (Step 6).

## Real architecture

```
customers_erp_master.csv → Copy_1
  ├→ Standardize NAME (rule MXNAME)             ──┐
  ├→ Standardize STREET (rule MXADDR)             ┤
  ├→ Standardize CITY+STATE_CODE+POSTAL_CODE       ┼→ Join_1→Join_2→Join_3→Join_4 → customers_erp_standardized.csv
  │   together (rule MXAREA)                       │
  ├→ Transformer_2: TELEPHONE_STD =                │
  │   Convert("-., #", "", TELEPHONE)            ──┤
  └→ Transformer_3: ZIP_STD =                       │
      Right("00000" : Trim(POSTAL_CODE), 5)       ──┘
```

Same pattern of 4 Joins chained on `ERP_ID` used in `JOB_02` with
`CUST_ID`.

## Key difference from the CRM: `POSTAL_CODE` survives as a pass-through, `TELEPHONE` doesn't

Unlike the CRM (where `PHONE_STD` and `ZIP_STD` were generated in a
separate job, `JOB_01`, as independent files), here both derived columns
were built inside the same job, off the same `Copy_1` branches:

- `POSTAL_CODE` reaches the flow **twice**: once as a pass-through inside
  the `CITY_STATE_POSTAL` branch (Standardize MXAREA, which takes the 3
  columns together), and once transformed into `ZIP_STD` in its own
  branch (`Transformer_3`). Both survive in the final output — this is
  the expected, correct behavior, not a bug.
- `TELEPHONE`, on the other hand, **only has one path in**: the
  `Transformer_2` branch. Since there's no Standardize branch for phone
  (no `MXPHONE` rule set exists on this instance — already confirmed at
  the start of the project), `TELEPHONE` doesn't survive as a separate
  column; only its transformed version, `TELEPHONE_STD`, exists.
  Expected and correct.

## Real problems found and solved

### 1. `POSTAL_CODE` inferred as INTEGER in the reader stage

**Anticipated symptom before running the job:** the reader
(`customers_erp_mastercsv_1`) inferred `POSTAL_CODE` as `INTEGER`, which
would have lost the leading zeros of Mexico City postal codes
(`"03100"`, `"06700"`) as early as the first stage — before the value
even reached the Transformer that generates `ZIP_STD`.

**Solution:** the type was fixed to `VARCHAR` directly in the reader
stage's schema, before running the job for the first time. Unlike the 6
earlier cases in the project (where the problem was caught *after* a
failed run), this time it was caught *before* execution, thanks to the
accumulated experience with the recurring pattern.

### 2. `ZIP_STD`/`TELEPHONE_STD` columns missing on the first attempt (output name not renamed)

**Symptom:** the first run finished "successful with warnings":
```
Join_4: Dropping component "POSTAL_CODE" because of a prior component with the same name.
```
and the output CSV had no `ZIP_STD` or `TELEPHONE_STD` column at all —
only `POSTAL_CODE` and `TELEPHONE` with already-clean values but the
original names.

**Cause:** in `Transformer_2` and `Transformer_3`, the output column had
been left with the same name as the input column (`TELEPHONE`/
`POSTAL_CODE`) instead of being renamed to `TELEPHONE_STD`/`ZIP_STD`. The
engine overwrote the value in the original column instead of creating a
new one, and when it reached `Join_4`, the duplicate copy of
`POSTAL_CODE` coming from the other 3 branches (inherited from `Copy_1`)
was silently dropped.

**Solution:** explicitly rename the output columns of both Transformers
to `ZIP_STD` and `TELEPHONE_STD`.

### 2b. Side effect: fixing `ZIP_STD`'s name completely lost the `TELEPHONE` branch

**Symptom:** in the following run, `ZIP_STD` already showed up correctly,
but neither `TELEPHONE` nor `TELEPHONE_STD` existed in the output CSV —
with no warning to flag it this time.

**Cause:** while editing `Transformer_3` (ZIP), the output link of
`Transformer_2` (phone) got disconnected from the `Join` that integrated
it into the main flow.

**Solution:** reconnect `Transformer_2`'s link to the corresponding Join
and confirm the column stayed selected in the output of every
intermediate Join up to the final file.

### 3. A "ghost" derivation: a column named `_STD` that's actually an untransformed pass-through — ⭐ new finding for the project

**Symptom:** after reconnecting the phone branch, `TELEPHONE_STD` already
showed up in the CSV, but for the only two records with dashes in the
original data (`ERP_ID 9007`: `"442-123-4567"`, `9008`:
`"999-222-3344"`), the `TELEPHONE_STD` value still had the dashes
uncleaned. The other 6 records looked "correct" only because they never
had dashes to clean in the first place — a false sense of success.

**Cause:** checking the "Output → Column mapping" tab of `Transformer_2`,
the `TELEPHONE_STD` column had, as its **Derivation**, literally
`TELEPHONE_STD.TELEPHONE` — that is, DataStage's standard
`link_name.column_name` notation for referencing an input column **with
no function wrapping it at all**. It's a pure pass-through disguised by
the output column's name (`TELEPHONE_STD`), which (incorrectly) suggested
cleanup logic had already been applied.

**Solution:** edit that column's derivation in the expression editor
(pencil icon next to the truncated field) and wrap the reference with the
real function:
```
Convert("-., #", "", TELEPHONE_STD.TELEPHONE)
```

**Lesson for the project:** the name of a Transformer's output column
(even one named `something_STD`) is **no guarantee** it has transformation
logic applied — you always have to open the expression editor and confirm
the expected function (`Convert`, `Right`, `Trim`, etc.) is actually
present, not just the name. Records with real edge cases in the data
(here, the 2 with dashes) are the only reliable proof that a cleanup
transformation actually runs — records that already came in "clean"
prove nothing.

## Final validated result

**8 records, 98 columns.** `ZIP_STD` and `TELEPHONE_STD` correct across
all 8, including the two dash edge cases:

| ERP_ID | NAME | ZIP_STD | TELEPHONE_STD |
|---|---|---|---|
| 9001 | Juan Perez Gomez | 03100 | 5512345678 |
| 9002 | Maria Fernanda Lopez Ruiz | 44100 | 3398765432 |
| 9003 | Carlos A. Sanchez | 64000 | 8122334455 |
| 9004 | Ana Torres Villegas | 72000 | 2221112233 |
| 9005 | Roberto Diaz Martinez | 06700 | 5588990011 |
| 9006 | Jose Luis Hernandez Paz | 37000 | 4775556677 |
| 9007 | Andrea Mendoza | 76000 | 4421234567 |
| 9008 | Diego Ramirez | 97000 | 9992223344 |

Real file at
`/outputs/erp_standardize_reports/customers_erp_standardized.csv`.

This dataset is Source B for Step 6 (Two-Source Match), comparable 1:1
against `customers_crm_clean.csv` via `ZIP_STD` ↔ `ZIP_STD` and
`TELEPHONE_STD` ↔ `PHONE_STD`.
