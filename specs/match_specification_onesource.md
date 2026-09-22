# Match Specification — One-Source Match (CRM deduplication)

> Real configuration, validated end to end. Asset: `onesource_match_crm`
> (type: DataStage match specification, match type: One-source
> independent).

## Blocking

| Name | Type | Data column |
|---|---|---|
| `Block_ZIP` | `CHAR - Character comparisons` | `ZIP_STD` |

## Match columns

| Name | Type | Data column | m-prob | u-prob | Param 1 |
|---|---|---|---|---|---|
| `Match_PrimerNombre` | `UNCERT` | `MatchPrimerNombre_MXNAME` | 0.9 | 0.01 | 0.2 |
| `Match_Apellido` | `UNCERT` | `MatchApellido_MXNAME` | 0.9 | 0.01 | 0.2 |
| `Match_Calle` | `UNCERT` | `MatchNombreCalle_MXADDR` | 0.9 | 0.01 | 0.2 |
| `Match_Telefono` | `CHAR` | `PHONE_STD` | 0.9 | 0.01 | — |

> **Important:** the comparison columns must be the `Match*` variants the
> rule set generates (optimized for comparison), not the display columns
> (`Apellido_MXNAME`, `Calle_MXADDR`) nor the internal keys (`*HashKey_*`,
> `*PackKey_*`).

## Cutoffs

| Threshold | Real value used | Where it's configured |
|---|---|---|
| Match | 12 | On the asset's pass, and again ("Override match cutoffs") on the job's stage |
| Clerical | 8 | Only on the job's stage — the asset designer doesn't expose this field |
| Overflow value | 10000 | On the asset's pass (required, don't leave blank) |

## Real result with the demo dataset

| Port | CUST_ID |
|---|---|
| Match (master) | 1001, 1003, 1006, 1010, 1014 |
| Duplicate (copy) | 1002, 1004, 1007, 1011, 1015 |
| Nonmatched | 1005, 1008, 1009, 1012, 1013 |
| Clerical | (none) |

Confirms exactly the 5 duplicate pairs seeded in
`docs/04-one-source-match.md` — full write-up there, including the
finding that the **Duplicate port is required** to avoid silently
losing 33% of the source records.
