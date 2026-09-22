# Specification — Standardize

> **Updated after validating on the real instance (Cloud Pak for Data /
> watsonx.data integration).** Unlike the first version of this document,
> **regional rule sets for Mexico do exist** under `Regions > Central
> America and Caribbean > Mexico`. There's no need to create custom rule
> sets or use generic `USNAME`/`USADDR` — use the following directly:

| Rule set found | Use |
|---|---|
| `MXNAME` | Person names (Mexico) |
| `MXADDR` | Addresses (Mexico) |
| `MXAREA` | City + State + Postal code (Mexico) |
| `MXPREP` | Name prepositions/particles (de, del, de la, etc. — supports MXNAME) |

## Name rule set — MXNAME

Input: `FULL_NAME`

Confirmed with the real Word/Character Investigate report on the demo
dataset — the patterns found were:

| Pattern detected | Real example | % of dataset |
|---|---|---|
| `F?` | "ANA TORRES" | 33% (5 records) |
| `F??` | "JUAN PEREZ GOMEZ" | 27% (4 records) |
| `FF?` | "Carlos Alberto Sanchez" | 20% (3 records) |
| `?F?I` | "Ma. Fernanda Lopez R." | 7% (1 record) — prefix + suffix |
| `F?I` | "SOFIA CASTILLO M." | 7% (1 record) — with suffix |
| `FI?` | "Jose L. Hernandez" | 7% (1 record) — with initial |

This confirms that `MXNAME` needs to resolve: prefixes ("Ma."), suffixes
("R.", "M."), and dotted initials ("L.") — exactly the cases seeded in the
test CSV.

## Address rule set — MXADDR

Input: `ADDRESS_LINE`

Real report (Word Investigate) — 11 distinct patterns over 15 records,
confirming street-type variants:

| Pattern | Real example |
|---|---|
| `T.??^` (13%, 2 records) | "Av. Insurgentes Sur 123" |
| `T??^` | "AV INSURGENTES SUR #123" |
| `T?^` | "Avenida Revolucion #1500" |
| `T.?^` (27%, 4 records) | "Av. Chapultepec 300" |
| `T?C^` | "Calzada Independencia No 77" |
| `T?^B.B` | "Calle Reforma 45 Col. Centro" |
| `T?C.^,B` | "Calle Reforma No. 45, Centro" |
| `T???^` | "Priv. de las Flores 12" |
| `T????^` | "Privada de las Flores #12" |
| `T^??^` | "Calle 5 de Mayo 88" |
| `TN^` | "Circuito Interior 500" |

## Area rule set — MXAREA (important finding!)

Input: **`CITY` + `STATE` + `ZIP` directly, as three separate columns**

> Unlike what was initially documented (which assumed `MXAREA` required a
> single combined column), **the Investigate/Standardize stage on this
> instance accepts all 3 columns selected together** and concatenates
> them internally for the analysis. This was confirmed because the output
> report shows the combined column as `"CITY+STATE+ZIP"`.
>
> **There's no need for a prior Transformer to concatenate manually.**
> Just select `CITY`, `STATE`, `ZIP` together when configuring the stage
> with the `MXAREA` rule.

Real report obtained:

| Pattern | Real example | % |
|---|---|---|
| `?D^` | "Guadalajara JAL 44100" | 33% (5 records) |
| `WWP?^` | "Cd. de Mexico CDMX 3100" | 27% (4 records) — note the 4-digit ZIP (lost the 0) |
| `DD^` | "Puebla PUE 72000" | 20% (3 records) |
| `WD^` | "Leon GTO 37000" | 13% (2 records) |
| `?P^` | "Toluca MEX 50000" | 7% (1 record) |

This confirms the expected finding of ZIPs with inconsistent length
("3100" instead of "03100") and the "Cd. de Mexico" variant vs. the rest
of the unabbreviated city names.

## Column with no rule set — PHONE

No phone rule set exists in this instance's rule list (`MXPHONE` doesn't
appear). It was resolved with a simple **Transformer stage**, already
proven and working correctly:

```
PHONE_STD = Convert("-., #", "", PHONE)   -- keeps digits only
```

Real confirmed output (`PHONE_STD.csv`) — correctly cleaned the dashes and
blanked out values that weren't real numbers (e.g. Patricia Ramírez's
field, which was originally just `","`).

## Column with no rule set — ZIP

No postal-code rule set exists either (`MXZIP` doesn't appear). Also
resolved with its own **Transformer**:

```
ZIP_STD = Right("00000" : Trim(ZIP), 5)   -- left zero-padding
```

⚠️ **Don't use `Str.PadLeft`** — it's not a valid function in DataStage's
expression language (see `docs/08-troubleshooting.md`, point 6).

⚠️ **Watch the output column's data type.** Even with the correct
expression, if the `ZIP_STD` column ends up typed as `INTEGER` in the
Transformer (or in any downstream Join that receives it), the engine
converts the string back to a number and the leading zero is lost again.
The type has to be checked and fixed in **every stage in the chain** — see
`docs/08-troubleshooting.md`, point 7.

## Update — the real Standardize (not just Investigate)

Everything above in this document was first validated with **Investigate**
stages (profiling). The **real Standardize** (which actually generates the
output `_STD` columns) was built afterward in `JOB_02_Standardize`, using
the same rules confirmed here. Full detail on that architecture, including
the 4 Joins that merge all branches by `CUST_ID`, in
[`docs/02-standardize.md`](../docs/02-standardize.md).

**Critical finding to avoid:** in the `FULL_NAME` Standardize stage, **do
not add the "Process All As Individual" literal** — it contaminates the
output last name (see `docs/08-troubleshooting.md`, point 5). The `MXNAME`
rule set assumes "individual" by default.

Final validated dataset: `/outputs/standardize_reports/customers_crm_standardized.csv`.
