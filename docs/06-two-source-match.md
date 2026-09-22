# Stage: Two-Source Match

> ⚠️ This is the **original theoretical plan**, written before the stage
> was actually built. See
> [`docs/06c-two-source-match-execution.md`](06c-two-source-match-execution.md)
> for the real configuration and everything that changed to get there.
> Kept here as a historical reference.

**Goal:** cross-check two different sources — the already-deduplicated
customer master (`customers_crm_clean`) and the ERP master
(`customers_erp_master.csv`) — to find out which customers already exist
in both systems, which are exclusive to each one, and which need manual
review.

## Preparation

1. **Standardize the ERP source** with the same stage/rules from Step 2
   (columns `NAME`, `STREET`, `CITY`, `STATE_CODE`, `POSTAL_CODE`,
   `TELEPHONE` → generate `_STD` equivalents to the CRM's). It's key to
   use the **same rule set** on both sources; otherwise the normalized
   columns won't be comparable.
2. Run **Match Frequency** (Step 3) again, this time over the combination
   of both standardized sources (or over each one separately, depending
   on your product version — check whether the stage supports combined
   frequencies).

## Stage configuration

1. Drag **Two-Source Match** from the *Data Quality* group.
2. Inputs:
   - **Source A (reference):** `customers_crm_clean` (standardized)
   - **Source B (comparison):** `customers_erp_master` (standardized)
   - Frequencies: Match Frequency output
3. Open the **Match Specification** (detail in
   [`specs/match_specification_twosource.md`](../specs/match_specification_twosource.md)):
   - **Blocking:** `STATE_STD` (both sources)
   - **Match columns:** `LAST_NAME_STD` (`UNCERT`), `FIRST_NAME_STD` (`UNCERT`),
     `STREET_NAME_STD` (`UNCERT`), `TELEPHONE_STD` (`CHAR`)
   - **Cutoffs:** Match = 13.0, Clerical = 9.0

## Output ports

| Port | Expected content with this dataset |
|---|---|
| **Match** | Juan Perez Gomez, Ma. Fernanda Lopez, Carlos Sanchez, Ana Torres, Roberto Diaz, Jose Luis Hernandez → exist in both systems |
| **Unmatched Source A** | Laura Gonzalez, Patricia Ramirez, Fernando Ortiz → CRM only (candidates to onboard into the ERP) |
| **Unmatched Source B** | Andrea Mendoza, Diego Ramirez → ERP only (candidates to sync into the CRM) |
| **Clerical** | Edge cases (e.g. the name matches but the address is quite different) → manual review queue |

## Closing the business case

The result of this step answers Marketing's original question directly:
*"which of our CRM customers are already registered in the ERP, and which
are genuinely new?"* — with full traceability of how each conclusion was
reached (profiling → standardization → deduplication → survivorship →
cross-system match).
