# Specification — Investigate

> **Validated on a real instance.** Character investigation had a UI bug on
> this instance (see `docs/08-troubleshooting.md`, point 2) — the
> configuration that worked end to end was **Word investigation** combined
> with Mexico's regional rule sets. This specification reflects that real
> configuration, not the initial theoretical one.

## Word Investigate — 4 parallel branches

| Branch | Input column(s) | Standardization rule | Goal |
|---|---|---|---|
| 1 | `FULL_NAME` | `MXNAME` | Detect prefixes ("Ma."), suffixes ("R.", "M."), initials ("L.") |
| 2 | `ADDRESS_LINE` | `MXADDR` | Detect street-type variants (Av/Avenida/Calz/Calzada/Priv/Privada) |
| 3 | `CITY` + `STATE` + `ZIP` (together) | `MXAREA` | Detect ZIPs with inconsistent length and city-name variants |
| 4 | `PHONE` | *(none — no phone rule set exists)* | Detect format with/without dashes and blank values |

## Real results obtained (see `/outputs/investigate_reports/`)

- **`FULL_NAME`**: 6 distinct patterns over 15 records — confirms the need
  to handle prefixes, suffixes, and initials in `MXNAME`.
- **`ADDRESS_LINE`**: 11 distinct patterns over 15 records — confirms
  street-type variants and house-number format (`#` vs `No.` vs direct)
  for `MXADDR`.
- **`CITY+STATE+ZIP`**: 5 distinct patterns — confirms ZIPs with
  inconsistent length (losing the leading zero) and the "Cd. de Mexico"
  variant vs. unabbreviated city names, for `MXAREA`.
- **`PHONE`**: not investigated with a rule set — resolved directly with a
  Transformer stage (`Convert("-., #", "", PHONE)`), confirmed to clean
  correctly to digits only and blank out values with no real digits.

These reports were the direct input used to confirm (not just design) the
Step 2 rules (the real Standardize).

