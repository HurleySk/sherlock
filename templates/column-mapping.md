# Column Mapping: {Entity Name}

## Entity Summary

| Field | Value |
|---|---|
| Source Table | `{dbo.SourceTable}` |
| Target Table/Entity | `{target_table}` |
| Pipeline/Job | `{pipeline_name}` |
| Key Strategy | `{match on alternate key, create or update}` |
| Record Count (approx) | {count} |

## Column Mapping

| Source Table | Source Column | Source Type | Target Table | Target Column | Target Type | Transform | Lookup Dependencies | Notes |
|---|---|---|---|---|---|---|---|---|
| {dbo.Source} | {SourceCol1} | {varchar(100)} | {target_table} | {target_col1} | {varchar(100)} | {LTRIM(RTRIM(SourceCol1))} | {None} | {Whitespace cleanup} |
| {dbo.Source} | {SourceCol2} | {int} | {target_table} | {target_col2} | {int} | {CASE SourceCol2 WHEN 1 THEN 100 END} | {None} | {Code mapping} |
| {dbo.Source} | {SourceFK} | {int} | {target_table} | {target_lookup} | {foreign key} | {JOIN ref_table ON FK = RefID} | {ref_table must be loaded first} | {FK resolution} |

## Transform Details

### {SourceCol2} Code Mapping

| Legacy Value | Target Value | Label |
|---|---|---|
| 1 | 100 | {Label A} |
| 2 | 200 | {Label B} |

### {Complex Transform Name}

{Describe the CTE, multi-table JOIN, or other complex logic here.}

## Lookup Dependencies

| Lookup Table | Source | Used By (Column) | Join Logic | Purpose |
|---|---|---|---|---|
| {ref_table} | {Legacy dbo.Ref} | {target_lookup} | {JOIN on FK = RefID} | {Resolve FK to target key} |

## Unmapped Columns

| Source Column | Reason |
|---|---|
| {LastModifiedDate} | {System-managed in target} |
| {InternalFlag} | {Not in scope per business decision} |

## Data Quality Notes

- {SourceCol1}: {N records exceed max length, will be truncated}
- {SourceFK}: {M records have NULL FK, lookup will resolve to NULL}
