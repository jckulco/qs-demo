# Job: JOB_08_MatchFrequency_TwoSource — ✅ COMPLETE AND VALIDATED

**Goal:** generate CRM and ERP frequencies over the match columns, as the
statistical input for the "Two-source" Match Specification in Step 6.

## ⚠️ Real design vs. the original design that was scrapped

The first attempt combined CRM + ERP into a single stream with a `Funnel`
before computing one shared frequency table (the same pattern as
`JOB_06_MasterMerge`). **This approach doesn't work here** and was
abandoned entirely — see finding #26 in `08-troubleshooting.md` for the
technical why. The final design uses **two fully independent branches**,
no Funnel:

```
customers_crm_clean.csv → Match_Frequency_3 → customers_crm_match_frequency.csv
  (columns: MatchPrimerNombre_MXNAME, MatchApellido_MXNAME,
            MatchNombreCalle_MXADDR, PHONE_STD, ZIP_STD)

customers_erp_standardized.csv → Transformer_1 (only renames 3 columns
  with an _ERP suffix; TELEPHONE_STD and ZIP_STD pass through untouched)
  → Match_Frequency_4 → customers_erp_match_frequency.csv
  (columns: MatchPrimerNombre_MXNAME_ERP, MatchApellido_MXNAME_ERP,
            MatchNombreCalle_MXADDR_ERP, TELEPHONE_STD, ZIP_STD)
```

Each frequency file carries **exactly the column names its corresponding
side uses in the Match Specification** — the Reference side (CRM) with no
suffix, the Data side (ERP) with an `_ERP` suffix on the 3 name/address
columns, and no renaming of the phone column (`TELEPHONE_STD`, not
`PHONE_STD`).

## Why the phone column is NOT renamed here (unlike an earlier attempt)

An intermediate attempt renamed `TELEPHONE_STD` → `PHONE_STD` in
`Transformer_1` to "unify" names between sources. This broke the whole
design: the Match Specification literally has `TELEPHONE_STD` as the
Data-side column name (never changed there), so the frequency file needed
an entry named `TELEPHONE_STD`, not `PHONE_STD`. The lesson: **column
names in the frequency file must exactly copy the names that appear in
the Match Specification**, column by column — not the "nicer" or
"unified" names that look cleaner.

## Final validated result

- `customers_crm_match_frequency.csv`: **142 records**, covers the 5
  columns on the Reference side.
- `customers_erp_match_frequency.csv`: **121 records**, covers the 5
  columns on the Data side.

Real files at `/outputs/match_frequency_twosource_reports/`.

## Connection in JOB_09_TwoSourceMatch

- `customers_erp_match_frequency.csv` → **`DataFreq`** port
- `customers_crm_match_frequency.csv` → **`RefFreq`** port

See the full write-up of the error trail (including the discovery that
the `Two-Source Match` stage needs 4 input links, not 2) in
`docs/06c-two-source-match-execution.md`.
