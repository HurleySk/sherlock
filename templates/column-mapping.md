# Column Mapping: {Entity Name}

## Entity Summary

| Field | Value |
|---|---|
| Source Table | `{dbo.SourceTable}` |
| Target DV Entity | `{alm_targetentity}` |
| Pipeline | `{pipeline_name}` |
| Alternate Key | `{alm_alternatekey}` |
| Upsert Behavior | Match on alternate key, create or update |
| Record Count (approx) | {count} |

## Column Mapping

| Source Table | Source Column | Source Type | DV Entity | DV Attribute | DV Type | Transform | Lookup Dependencies | Notes |
|---|---|---|---|---|---|---|---|---|
| {dbo.Source} | {SourceCol1} | {varchar(100)} | {alm_entity} | {alm_attribute1} | {Single Line of Text} | {LTRIM(RTRIM(SourceCol1))} | {None} | {Whitespace cleanup} |
| {dbo.Source} | {SourceCol2} | {int} | {alm_entity} | {alm_attribute2} | {Choice} | {CASE SourceCol2 WHEN 1 THEN 455780000 END} | {None} | {Option set mapping} |
| {dbo.Source} | {SourceFK} | {int} | {alm_entity} | {alm_lookupfield} | {Lookup} | {JOIN stg_Ref ON FK = RefID, resolve to DV GUID} | {stg_Ref must be loaded first} | {FK resolution} |

## Transform Details

### {SourceCol2} Option Set Mapping

| Legacy Value | DV Option Set Value | Label |
|---|---|---|
| 1 | 455780000 | {Label A} |
| 2 | 455780001 | {Label B} |

### {Complex Transform Name}

{Describe the CTE, multi-table JOIN, or other complex logic here.}

## Lookup Dependencies

| Lookup Table | Source | Used By (Column) | Join Logic | Purpose |
|---|---|---|---|---|
| {stg_Ref} | {Legacy dbo.Ref} | {alm_lookupfield} | {JOIN on FK = RefID} | {Resolve FK to DV GUID} |

## Unmapped Columns

| Source Column | Reason |
|---|---|
| {LastModifiedDate} | {System-managed in Dataverse} |
| {InternalFlag} | {Not in scope per business decision} |

## Data Quality Notes

- {SourceCol1}: {N records exceed max length, will be truncated}
- {SourceFK}: {M records have NULL FK, lookup will resolve to NULL}
