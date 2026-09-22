# Stage: Investigate

**Goal:** profile the quality of the source data before touching it,
identifying patterns, outliers, and inconsistent formats.

> **Important correction (after testing on a real instance):** **Character
> investigation** mode had a UI bug in the column selector (checkboxes
> couldn't be checked — see [`08-troubleshooting.md`](08-troubleshooting.md),
> point 2). The path that **did work end to end was Word Investigation
> combined with Mexico's regional rule sets**, and that's the configuration
> documented below. Also, instead of investigating all 6 columns in a
> single stage, **4 parallel branches** were built, one per column group,
> each with its own standardization rule attached right from the
> Investigate stage.

## Real configuration used (validated on the instance)

1. Drag **Investigate** from the *Data Quality* group in the palette (one
   per branch).
2. Connect the input to the output of `customers_crm.csv` (via an
   intermediate `Copy` stage that fans the flow out to the 4 branches).
3. Investigation type: **Word** (not Character — see note above).
4. Configure each branch like this:

| Branch | Selected column(s) | Standardization rule applied |
|---|---|---|
| 1 | `ADDRESS_LINE` | `MXADDR` |
| 2 | `FULL_NAME` | `MXNAME` |
| 3 | `CITY` + `STATE` + `ZIP` (all 3 together, in the same stage) | `MXAREA` |
| 4 | `PHONE` | *(no rule set — see PHONE note below)* |

> **About branch 3:** there was no need to concatenate `CITY`+`STATE`+`ZIP`
> with a Transformer beforehand — the stage accepted the 3 selected
> columns directly and concatenated them internally for the analysis (the
> output report shows them as `"CITY+STATE+ZIP"`). See finding #4 in
> `08-troubleshooting.md`.

> **About `PHONE`:** no phone rule set (`MXPHONE`) exists on the tested
> instance. This branch **doesn't use Investigate with a rule** — instead
> it uses a separate **Transformer stage**, with the expression:
> ```
> PHONE_STD = Convert("-., #", "", PHONE)
> ```
> which cleans the field down to digits only. This branch is left out of
> the main flow and needs to be re-joined (Join/Funnel) before Match
> Frequency — see the pending item in `docs/00-step-by-step-guide.md`.

5. Output of each Investigate branch: a **frequency/pattern report** — a
   table with word pattern, sample, count, and %. The 4 real reports
   generated are in `/outputs/investigate_reports/`.

## What the real reports showed

| Branch | Pattern that shows up | Insight |
|---|---|---|
| `FULL_NAME` | Patterns `F?`, `FF?`, `FI?`, `?F?I` — a mix of simple name, compound name, with initial ("L."), with prefix ("Ma.") and suffix ("R.", "M.") | Confirms `MXNAME` needs to resolve prefixes, suffixes, and initials |
| `ADDRESS_LINE` | 11 distinct patterns over 15 records: "Av." / "AV" / "Avenida" / "Calle" / "Calzada" / "Priv." / "Privada", "#" vs "No." vs direct | Confirms `MXADDR` needs to normalize street type and number format |
| `CITY+STATE+ZIP` | 5 patterns: 4-digit ZIP (losing the leading zero, e.g. "3100"), "Cd. de Mexico" vs unabbreviated cities | Confirms `MXAREA` needs to resolve ZIP padding and the city-name variant |
| `PHONE` (via Transformer) | With dashes, without dashes, blank, or symbols only (e.g. `","`) | Confirmed: `Convert()` correctly cleans down to digits only, and blanks out values with no real digits |

This report is the direct input used to design/confirm the Step 2 rules
(the real Standardize). The Investigate stages used here **only generate
the pattern report** — they don't yet generate the output `_STD` columns
(that's done by the actual Standardize stage, in Step 2).
