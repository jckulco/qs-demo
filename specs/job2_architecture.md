# Specification — JOB_02_Standardize architecture

Exact diagram of the validated job, so it can be reproduced unambiguously.

## Job 1 (helper branches, inside JOB_01_Investigate_Standardize_Match)

```
customers_crmcsv_1 → Copy_2 ─┬─ Investigate(Word) ADDRESS_LINE + MXADDR → ADDRESS_LINE_frequency_report.csv
                              ├─ Investigate(Word) FULL_NAME + MXNAME     → FULL_NAME_frequency_report.csv
                              ├─ Investigate(Word) CITY,STATE,ZIP + MXAREA→ CITY_STATE_ZIP_frequency_report.csv
                              ├─ Transformer PHONE  (CUST_ID pass-through, PHONE_STD=Convert(...)) → PHONE_STD.csv
                              └─ Transformer ZIP    (CUST_ID pass-through, ZIP_STD=Right(...))      → ZIP_STD.csv
```

> Only the two Transformer branches (`PHONE`, `ZIP`) feed into Job 2. The 3
> Investigate branches are diagnostic-only and don't connect to anything
> else.

## Job 2 (JOB_02_Standardize) — real topology

```
customers_crmcsv_1 → Copy_1 ─┬─ Standardize FULL_NAME (MXNAME)      → Link_2 ─┐
                              ├─ Standardize ADDRESS_LINE (MXADDR)   → Link_3 ─┼→ Join_1
                              └─ Standardize CITY,STATE,ZIP (MXAREA) → Link_5 ─┘        │
                                                                                          ▼
                                                                                       Join_1 → Link_4 ─┐
                                                                                                          ├→ Join_2
                                                              (CITY_STATE_ZIP) → Link_6 ─────────────────┘
                                                                                          │
                                                                                          ▼
                                                                                       Join_2 → Link_9 ─┐
                                                                                                          ├→ Join_3
                                                      PHONE_STDcsv_1 → Link_7 ───────────────────────────┘
                                                                                          │
                                                                                          ▼
                                                                                       Join_3 → Link_11 ─┐
                                                                                                           ├→ Join_4
                                                       ZIP_STDcsv_1 → Link_13 ────────────────────────────┘
                                                                                          │
                                                                                          ▼
                                                                            customers_crm_standardized.csv
```

## Configuration of each Join

All 4 Joins use the same base configuration:

| Join | Left input | Right input | Key | Type |
|---|---|---|---|---|
| `Join_1` | `FULL_NAME` (Standardize) | `ADDRESS_LINE` (Standardize) | `CUST_ID` = `CUST_ID` | Inner |
| `Join_2` | `Join_1` | `CITY_STATE_ZIP` (Standardize) | `CUST_ID` = `CUST_ID` | Inner |
| `Join_3` | `Join_2` | `PHONE_STDcsv_1` (file) | `CUST_ID` = `CUST_ID` | Inner |
| `Join_4` | `Join_3` | `ZIP_STDcsv_1` (file) | `CUST_ID` = `CUST_ID` | Inner |

**Checklist before running each Join:**
- [ ] Both inputs have `CUST_ID` in their output schema.
- [ ] The `CUST_ID` data type matches on both sides (Integer with Integer, not mixed with string).
- [ ] Check the Join's own output-column tab — it can silently redefine types on its own (see `docs/08-troubleshooting.md`, point 7, for the real case that happened with `ZIP_STD`).

## Final output

- File: `customers_crm_standardized.csv`
- 97 columns, 15 rows
- Includes: original columns from `customers_crm.csv` + decomposed columns from `MXNAME`/`MXADDR`/`MXAREA` + `PHONE_STD` + `ZIP_STD`
- Available in this repo: `/outputs/standardize_reports/customers_crm_standardized.csv`
