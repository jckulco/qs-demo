# Match Specification — Two-Source Match (clean CRM vs ERP)

> ⚠️ This is the **original theoretical plan**, written before the stage
> was actually built. The real, validated configuration ended up
> different in several important ways (real column names with an `_ERP`
> suffix, `ZIP_STD` as the blocking column instead of `STATE_STD`, a Match
> cutoff of 11 instead of 13.0). See
> [`docs/06c-two-source-match-execution.md`](../docs/06c-two-source-match-execution.md)
> for the real configuration and everything that had to change to get
> there. This file is kept as a historical reference.

## Sources

- **Source A (reference):** `customers_crm_clean` (Survive output)
- **Source B (comparison):** `customers_erp_master` (standardized)

## Blocking

| Column | Reason |
|---|---|
| STATE_STD | Same criterion as One-Source Match |

## Match columns

| Column A | Column B | Algorithm |
|---|---|---|
| LAST_NAME_STD | LAST_NAME_STD | UNCERT |
| FIRST_NAME_STD | FIRST_NAME_STD | UNCERT |
| STREET_NAME_STD | STREET_NAME_STD | UNCERT |
| TELEPHONE_STD | TELEPHONE_STD | CHAR |

## Cutoffs

| Threshold | Suggested initial value |
|---|---|
| Match cutoff | 13.0 |
| Clerical cutoff | 9.0 |

## Expected result

| Match (exist in both) | CRM only | ERP only |
|---|---|---|
| Juan Perez Gomez | Laura Gonzalez | Andrea Mendoza |
| Maria Fernanda Lopez | Patricia Ramirez Cruz | Diego Ramirez |
| Carlos Alberto Sanchez | Fernando Ortiz | |
| Ana Torres | | |
| Roberto Diaz Martinez | | |
| Jose Luis Hernandez | | |
