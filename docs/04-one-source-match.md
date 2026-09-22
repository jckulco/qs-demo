# Stage: One-Source Match (Unduplicate Match)

**Goal:** find duplicates **within a single source** (the CRM itself),
using a real Match Specification built in the platform's dedicated
designer.

> **Validated on a real instance** in `JOB_04_OneSourceMatch`, with a
> Match Specification (`onesource_match_crm`) built from scratch. This
> page documents the full process, including 3 findings that weren't
> obvious from the public documentation.

## Step A — Build the Match Specification (outside the job)

The Match Specification **isn't created from inside the job** — it's a
standalone project asset:

1. From the project (not from the job), go to **"New asset" → search
   "datastage"** → **"Create reusable DataStage components"**.
2. Select the **"Match specification"** type.
3. Name: `onesource_match_crm`. Match type: **"One-source" → "One-source
   independent"** (compares every record against every other one in the
   same block, without relying on order — the most thorough option for a
   dataset this small; "dependent" is more efficient for huge datasets
   but can miss some matches).
4. In the designer that opens, **"Add input schema"** → Browse → select
   `customers_crm_standardized.csv` (this imports the 97 columns as
   columns available to the specification).
5. **"Add pass"** → this creates `Match_Pass_1`.
6. **"Edit pass"** → right panel → next to "Match specification columns,"
   click **"Edit."**

### Blocking columns

| Name | Type | Data column |
|---|---|---|
| `Block_ZIP` | `CHAR - Character comparisons` | `ZIP_STD` |

⚠️ **Requires `ZIP_STD` to be typed as text in the imported schema.** If it
shows up as `INTEGER`, the engine rejects the blocking with the error:
```
ERROR IIS-QSEE-PMVL-00009: Input column ZIP_STD is a numeric column.
Numeric columns cannot be used for character blocking
```
Fix the type in the source stage before continuing (see
`docs/08-troubleshooting.md`).

### Match columns

| Name | Type | Data column | m-prob | u-prob | Param 1 |
|---|---|---|---|---|---|
| `Match_PrimerNombre` | `UNCERT` | `MatchPrimerNombre_MXNAME` | 0.9 | 0.01 | 0.2 |
| `Match_Apellido` | `UNCERT` | `MatchApellido_MXNAME` | 0.9 | 0.01 | 0.2 |
| `Match_Calle` | `UNCERT` | `MatchNombreCalle_MXADDR` | 0.9 | 0.01 | 0.2 |
| `Match_Telefono` | `CHAR` | `PHONE_STD` | 0.9 | 0.01 | *(not applicable)* |

> **`Param 1` on `UNCERT`** is the character similarity threshold (0 to
> 1). `0.2` is the reference value used in QualityStage for name
> comparison — permissive enough to tolerate spelling variants without
> becoming too loose.
>
> **Don't check "Vectors" or "Reverse"** — they don't apply here
> (single-value-per-record columns, left-to-right comparison).

### Cutoffs

- **Match:** `12`
- **Overflow value:** `10000` (default — don't clear it, it's required to save)
- The **Clerical cutoff doesn't appear on this designer panel** — it's
  configured later, at the job's stage level (see Step C).

7. Save the pass.
8. **Very important:** go to this asset's **"Provision"** tab and run it —
   this **publishes** the specification. Without this step, the Match
   Specification picker in the job's stage shows "No assets found" even
   though the asset already exists in the project. Expected confirmation:
   `"Publish successful"`.

## Step B — Build the job

```
customers_crm_standardizedcsv_1 ──┐
                                    ├→ Onesource_Match_1 ─┬→ Match     → onesource_match_matched.csv
customers_crm_match_frequencycsv_1┘                       ├→ Clerical  → onesource_match_clerical.csv
                                                           ├→ Duplicate → onesource_match_duplicate.csv
                                                           ├→ Nonmatched→ onesource_match_nonmatched.csv
                                                           └→ Match statistics → onesource_match_statistics.csv
```

## Step C — Configure the `Onesource_Match_1` stage

1. **Match type:** `Independent` (matches what was chosen in the asset).
2. **Match specification:** Browse → `onesource_match_crm` (only shows up
   after "Provision" was run in Step A).
3. **Match outputs** — check the ones you need materialized as a file:
   - ☑️ Match
   - ☑️ Clerical
   - ☑️ **Duplicate** — see the critical finding below
   - ☑️ Nonmatched
   - ☑️ Match statistics
4. **Override match cutoffs** → check it to be able to edit the cutoff
   table (without it, the table stays read-only). Set:
   - Match: `12`
   - Clerical: `8`

## ⚠️ Critical finding: the "Duplicate" port is required

In the first run, **without checking "Duplicate,"** the result was:
- Match: 5 records
- Nonmatched: 5 records
- Total: 10 of 15 — **5 records missing with no apparent explanation**.

The 5 missing ones were exactly **the duplicate copies** of each group
(`1002`, `1004`, `1007`, `1011`, `1015`). The matching engine designates
**one "master" record** per duplicate group (which goes out the `Match`
port) and sends the other copies of the same group to the **`Duplicate`**
port — a separate port that, if not checked and connected, **silently
drops those records with no warning or error at all**.

**If your goal is to account for 100% of your source records** (not just
see "which are duplicates of what"), **the Duplicate port is required**,
not optional.

## Final validated result

| Port | Records | CUST_ID |
|---|---|---|
| **Match** (masters) | 5 | 1001, 1003, 1006, 1010, 1014 |
| **Duplicate** (copies) | 5 | 1002, 1004, 1007, 1011, 1015 |
| **Nonmatched** (unique) | 5 | 1005, 1008, 1009, 1012, 1013 |
| **Clerical** | 0 | — (data already well normalized, no ambiguous cases) |
| **Match statistics** | 19 | — (score distribution) |

**15 of 15 records accounted for.** The 5 duplicate pairs seeded from the
demo's original design were detected exactly:

| SetID | Match (master) | Duplicate (copy) |
|---|---|---|
| 1 | Juan Perez Gomez (1001) | JUAN PEREZ GOMEZ (1002) |
| 3 | Maria Fernanda Lopez (1003) | Ma. Fernanda Lopez R. (1004) |
| 6 | Ana Torres (1006) | ANA TORRES (1007) |
| 10 | Jose Luis Hernandez (1010) | Jose L. Hernandez (1011) |
| 14 | Sofia Castillo (1014) | SOFIA CASTILLO M. (1015) |

Real files in `/outputs/onesource_match_reports/`.

## On the (by-now familiar) leading-zero ZIP problem

This was the **fourth time** in the project that `ZIP_STD` lost its
leading zero due to incorrect typing — this time in the schema of the
match stage's own output `Sequential file`. Fixed with the same procedure
as always: check and correct the type to `VARCHAR` at every point in the
chain. Confirmed in the final result:
```
1001 | zip= 03100  ✅
1008 | zip= 06700  ✅
1012 | zip= 06140  ✅
```
See the full pattern documented in `docs/08-troubleshooting.md`, point 7.
