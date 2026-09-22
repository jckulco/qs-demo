# Expected result — validation checklist

> Updated with the **real, validated** numbers from closing out the
> project (not the original theoretical projection). If you're
> reproducing this demo, your results should match these exactly.

## After One-Source Match + Survive

From 15 original records → **10 unique records**:

- Survivor of group {1001, 1002} → 1 record (Juan Perez Gomez)
- Survivor of group {1003, 1004} → 1 record (Maria Fernanda Lopez)
- Survivor of group {1006, 1007} → 1 record (Ana Torres)
- Survivor of group {1010, 1011} → 1 record (Jose Luis Hernandez)
- Survivor of group {1014, 1015} → 1 record (Sofia Castillo)
- No duplicate: 1005, 1008, 1009, 1012, 1013 (5 records)

Total: 5 + 5 = **10 records** in `customers_crm_clean.csv`
(`outputs/master_clean_reports/`).

## After Two-Source Match against the ERP

- **6 matches:** Juan Perez Gomez, Maria Fernanda Lopez, Carlos Alberto
  Sanchez, Ana Torres, Roberto Diaz Martinez, Jose Luis Hernandez.
- **4 CRM-only (Reference nonmatched):** Laura Gonzalez, Patricia
  Ramirez Cruz, Fernando Ortiz, **Sofia Castillo**.
- **2 ERP-only (Data nonmatched):** Andrea Mendoza, Diego Ramirez.
- **Clerical: 0.**

Real files in `outputs/twosource_match_reports/`.

### Note on Sofía Castillo

The original theoretical plan (`docs/06-two-source-match.md`) assumed
only 3 "CRM-only" records, expecting the ERP to have a row for Sofía
Castillo. The real `customers_erp_master.csv` used in this demo has 8
rows, not 9 — it never included Sofía. The real result (4, not 3) is
correct against the real data in this repo; if you clone this project as
is, your 4 records should match the ones above exactly.

## If your numbers don't match

1. Check the **Match Specification cutoffs** first — in this demo, the
   Match cutoff for Two-Source Match had to be lowered from 13.0 (initial
   theoretical value) to **11** for the 6 real pairs to land in "Match"
   instead of "Clerical." See `docs/06c-two-source-match-execution.md`,
   finding #30.
2. Confirm that the `ZIP_STD` and phone columns arrive as `VARCHAR` at
   every stage — it's the most frequent recurring problem in the project
   (see `docs/08-troubleshooting.md`, points 15, 20, 25, 26).
3. If you use Standardize rule sets other than `MXNAME`/`MXADDR`/`MXAREA`
   (for example `USNAME`/`USADDR` for the US), the match weights computed
   by the engine will differ from this demo's — adjust the cutoffs based
   on what you observe in `qsMatchWeight`, don't copy the values here
   directly.
