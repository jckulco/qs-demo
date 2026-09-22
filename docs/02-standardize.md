# Stage: Standardize

**Goal:** turn the raw data into a consistent format, decomposing free-text
fields (name, address, area) into normalized components, and producing a
single clean dataset (`customers_crm_standardized.csv`) ready for Match
Frequency.

> **This chapter documents the real architecture, validated end to end**
> across two jobs (`JOB_01_Investigate_Standardize_Match` and
> `JOB_02_Standardize`), including the 4 real problems solved along the
> way. If you're reproducing this for the first time, read
> [`08-troubleshooting.md`](08-troubleshooting.md) first — it'll save you
> a few round trips.

## Final architecture

The real Standardize ended up split across **two jobs**:

### Job 1 — generates 2 helper files (via Transformer, no rule set)

Inside `JOB_01_Investigate_Standardize_Match`, two extra branches besides
the Investigate ones:

| Branch | Stage | Expression | Output |
|---|---|---|---|
| PHONE | Transformer | `CUST_ID = CUST_ID` (pass-through) · `PHONE_STD = Convert("-., #", "", PHONE)` | `PHONE_STD.csv` |
| ZIP | Transformer | `CUST_ID = CUST_ID` (pass-through) · `ZIP_STD = Right("00000" : Trim(ZIP), 5)` | `ZIP_STD.csv` |

Both files are uploaded as project assets and read in Job 2 as
independent sources.

### Job 2 — the real Standardize + merging everything by `CUST_ID`

```
customers_crmcsv_1 → Copy_1 ─┬─ Standardize FULL_NAME (rule MXNAME)      ──┐
                              ├─ Standardize ADDRESS_LINE (rule MXADDR)    ──┤
                              └─ Standardize CITY+STATE+ZIP (rule MXAREA)  ──┤
                                                                              ├─ Join_1 → Join_2 → Join_3 → Join_4 → customers_crm_standardized.csv
                              PHONE_STD.csv (file, already clean) ───────────┤
                              ZIP_STD.csv (file, already clean)  ────────────┘
```

- **3 Standardize stages** (one per rule set), each connected from `Copy_1`.
- **2 external sources** (`PHONE_STDcsv_1`, `ZIP_STDcsv_1`) — the CSVs generated in Job 1.
- **4 chained Joins**, all keyed on `CUST_ID`, merging the 5 branches into one record per customer.
- Final output: `customers_crm_standardized.csv` (97 columns, 15 rows).

## Configuration of each Standardize

### `FULL_NAME` → rule `MXNAME`

- Column: `FULL_NAME` (added via "New columns +").
- **Don't add the "Process All As Individual" literal** — see
  [problem 1 in troubleshooting](08-troubleshooting.md). The rule set
  assumes "individual" by default; adding the literal concatenated
  `ZQPINDZQ` directly onto the output last name.

### `ADDRESS_LINE` → rule `MXADDR`

- Column: `ADDRESS_LINE`.
- No complications — worked as expected from the very first test.

### `CITY` + `STATE` + `ZIP` → rule `MXAREA`

- The **3 columns together**, selected in the same stage (not one at a
  time). The stage concatenates them internally — confirmed because the
  output report shows them as `"CITY+STATE+ZIP"`.
- **Real limitation found:** the `CodigoPostal_MXAREA` field this rule
  generates doesn't handle ZIP padding — it comes out as a placeholder
  `"00000"` for all 15 records. That's why the correct ZIP comes from the
  Job 1 Transformer (`ZIP_STD.csv`) instead of this rule.

## Merging the 5 branches — the 4 Joins

Each Join uses `CUST_ID = CUST_ID` as the key, type **Inner join**:

1. `Join_1`: `FULL_NAME` (Standardize) + `ADDRESS_LINE` (Standardize)
2. `Join_2`: previous result + `CITY_STATE_ZIP` (Standardize)
3. `Join_3`: previous result + `PHONE_STDcsv_1` (file)
4. `Join_4`: previous result + `ZIP_STDcsv_1` (file) → final output

> **Critical point:** every branch must pass `CUST_ID` in its output
> schema — no exceptions, including the Standardize stages (which by
> default only emit the rule's decomposed columns, not the original
> ones). Always check the **Output** tab of each stage before connecting
> it to a Join.

## The leading-zero ZIP problem — solved across 3 layers

This was the longest bug to track down in the project. The leading `0` in
the ZIP (`"3100"` instead of `"03100"`) was getting lost at **three
different points** in the chain, and each one had to be fixed separately:

1. **Source column `ZIP`** — arrived typed as `INTEGER` from the CSV
   connector. An `INTEGER` can never have a leading zero.
2. **Output column `ZIP_STD` in the Transformer** (Job 1) — even though
   the expression (`Right("00000" : Trim(ZIP), 5)`) generated the correct
   string, the Transformer's output column was still typed `INTEGER` by
   default, and the engine converted the string back to a number at the
   end, losing the zero again.
3. **Output column `ZIP_STD` in `Join_4`** (Job 2) — even with the source
   file already fixed (confirmed as `string[variable_max=5]`), the
   `Join_4` stage had its own output schema defined as `int32`, and the
   log showed the explicit warning:
   ```
   WARNING ... Implicit conversion from source type "string[variable_max=5]"
   to result type "int32": Converting string to number.
   ```

**Lesson:** a column's data type can be redefined at **every stage it
passes through**, regardless of how it's typed at the source. You have to
check the output schema at every link in the chain (Transformer → file →
reader connector → Join), not assume that fixing the source is enough.

Full detail on each layer in
[`08-troubleshooting.md`](08-troubleshooting.md).

## Final validated output

| Field | Result |
|---|---|
| `Apellido_MXNAME` | Clean, no literal contamination |
| `ADDRESS_LINE` via `MXADDR` | Decomposed correctly |
| `CITY_STATE_ZIP` via `MXAREA` | Runs, but `CodigoPostal_MXAREA` isn't reliable — use `ZIP_STD` instead |
| `PHONE_STD` | Clean, correct `NULL` where there was no phone |
| `ZIP_STD` | Correct, with leading-zero padding |
| Record 1004 ("Ma. Fernanda Lopez R.") | Parses, though with the "R." suffix stuck onto the last name — a known limitation of the rule set with this pattern, not a blocker |

Real file: `outputs/standardize_reports/customers_crm_standardized.csv`.
