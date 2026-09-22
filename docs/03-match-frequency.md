# Stage: Match Frequency

**Goal:** precompute how often each value appears in the columns that
will be used as the match key, so the matching engine can properly weigh
rare vs. common matches in the next step.

> **Validated on a real instance** in `JOB_03_MatchFrequency`. Below is
> the exact configuration that worked, including the correct column to
> pick when a rule set generates multiple variants of the same base
> column.

## Real configuration used

1. Source: `customers_crm_standardized.csv` (the clean dataset from Step 2).
2. Stage: **Match Frequency**.
3. **"Do not use a match specification"** → checked (at this point in the
   flow, the Match Specification doesn't exist yet — that's built in
   Step 4).
4. Columns selected as key — **use the `Match*` columns the rule sets
   generate, not the base ones**:

| Column selected | Why this one and not another |
|---|---|
| `MatchPrimerNombre_MXNAME` | Comparison-optimized name version (not the display one) |
| `MatchApellido_MXNAME` | Same, for last name |
| `MatchNombreCalle_MXADDR` | Same, for street. **Don't use** `Calle_MXADDR` (display) or `MatchNombreCalleHashKey_MXADDR`/`MatchNombreCallePackKey_MXADDR` (internal engine keys — never selected manually) |
| `PHONE_STD` | Already-cleaned phone |
| `ZIP_STD` | Already-cleaned postal code, correctly padded |

5. Save the stage, run the job.
6. Output: `customers_crm_match_frequency.csv` — **not a customer-data
   CSV**, it's a specialized internal structure (`qsFreqValue`,
   `qsFreqCounts`, `qsFreqColumnID`, `qsFreqHeaderFlag`) that the matching
   engine consumes in the next step. Opening it you'll see things like:
   ```
   "ANA","0000000002 0000000000 0000000000 ","1","1"
   ```
   where `"ANA"` with a count of `2` correctly reflects that this name
   appears twice in the dataset (from the seeded Ana Torres duplicate).

## Note on "First line is column name"

⚠️ Remember to enable this option when configuring the output
`Sequential file` — otherwise, when this file is read in the next job,
column names don't import correctly (see `docs/08-troubleshooting.md`).

## Next step

This file (`customers_crm_match_frequency.csv`) is connected as the
second input of the **One-Source Match** stage in Step 4, alongside the
standardized dataset. See
[`docs/04-one-source-match.md`](04-one-source-match.md).
