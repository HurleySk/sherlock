---
name: lineage-tracer
description: "How to trace data lineage through ETL pipelines from source to sink with column-level precision"
---

# Lineage Tracer

## The Three Hops

Every migrated column makes three hops:

1. **Source** (legacy SQL table) -- copied via pipeline Copy activity to...
2. **Staging** (Azure SQL table) -- transformed via embedded SQL query to...
3. **Sink** (Dataverse entity) -- upserted via pipeline Copy activity

The goal of lineage tracing is to follow each column through all three hops and document what happens at each step.

## How to Trace: ADF Pipeline-Based Migrations

### Step 1: Find the Pipeline

Look in `work-repo/pipeline/` for the Load/migration pipeline. Pipeline names typically follow patterns like:
- `onprem_Nightly{Entity}Load`
- `PL_{Entity}_Migration`
- `Load_{Entity}`

### Step 2: Identify Copy Activities

Each Copy activity in the pipeline has a `source` and `sink` section in its JSON definition. A typical migration pipeline has two Copy activities:
- **Copy 1**: Source (on-prem) to Staging (Azure SQL)
- **Copy 2**: Staging (Azure SQL) to Sink (Dataverse)

Some pipelines combine these or add intermediate steps. Read the activity dependencies to understand the execution order.

### Step 3: Trace Hop 1 (Source to Staging)

In the first Copy activity:
- **Source dataset** references an on-prem linked service and table name. This is the legacy source.
- **`sqlReaderQuery`** or table reference tells you which source columns are selected. If there is no query, all columns from the source table are copied.
- **Sink** points to the staging table in Azure SQL.
- **`preCopyScript`** typically contains `TRUNCATE TABLE stg_TableName` -- this confirms the staging table name.

What to extract:
- Source table name and columns
- Staging table name
- Any column filtering or renaming in this hop (usually minimal -- most transforms happen in Hop 2)

### Step 4: Trace Hop 2 (Staging to Sink)

In the second Copy activity:
- Look for the activity that reads FROM staging and writes TO Dataverse.
- The **`sqlReaderQuery`** contains the transform SQL. This is where the real work happens.

Extract from the SQL query:
- **Column aliases**: `stg.OrgName AS alm_name` maps staging column to output column name
- **JOINs**: Lookup resolution joins (e.g., `LEFT JOIN stg_Lookup ON stg.FK = stg_Lookup.ID`) resolve legacy FKs to DV GUIDs
- **CASE statements**: Enum/option set conversion (`CASE TypeCode WHEN 1 THEN 455780000 END`)
- **String functions**: `LTRIM(RTRIM(Name))`, `SUBSTRING(Code, 1, 10)`, `REPLACE(Phone, '-', '')`
- **COALESCE**: Fallback chains for nullable fields
- **Computed columns**: `CONCAT(FirstName, ' ', LastName)`, arithmetic, conditional logic
- **CTE chains**: Multi-step transforms common in document metadata or hierarchies
- **WHERE clauses**: Filtering (e.g., `WHERE IsActive = 1`)

### Step 5: Trace Hop 3 (Write to Dataverse)

In the sink configuration of the second Copy activity:
- **Entity name**: The Dataverse entity logical name or entity set name
- **Write behavior**: Upsert or append
- **Alternate key**: Used for matching existing records (determines create vs update)
- **Write batch size**: How many records per API call
- **Bypass plugin execution**: Whether plugins/workflows fire during upsert

## How to Trace: SP-Based Migrations

For stored procedure-based migrations:

1. **Find the SP** in `work-repo/SQL DB/.../Stored Procedures/` or `db-export/{connection}/procedure/`
2. **Read the SELECT statement**: Source columns are in the FROM/JOIN clauses, output columns are the SELECT aliases
3. **Follow the same three-hop logic**: The SP embodies Hop 2 (the transform), so trace backward to find the source (Hop 1) and forward to find the sink (Hop 3)

## Common Transform Patterns

### CASE Statements (Option Set Mapping)
```sql
CASE src.StatusCode
    WHEN 'A' THEN 455780000  -- Active
    WHEN 'I' THEN 455780001  -- Inactive
    ELSE NULL
END AS alm_status
```
Document: Legacy code, DV option set value, and label.

### LEFT JOIN (Lookup Resolution)
```sql
LEFT JOIN stg_BusinessUnit bu ON src.BUID = bu.LegacyBUID
-- Output: bu.alm_businessunitid AS alm_parentbusinessunitid
```
Document: Which lookup table, join columns, and which DV GUID field it resolves to.

### String Cleanup
```sql
LTRIM(RTRIM(REPLACE(src.Name, CHAR(9), '')))
```
Document: What cleanup is applied and why.

### CTE Chains
```sql
WITH Hierarchy AS (
    SELECT ID, ParentID, Name, 0 AS Level FROM Org WHERE ParentID IS NULL
    UNION ALL
    SELECT o.ID, o.ParentID, o.Name, h.Level + 1 FROM Org o JOIN Hierarchy h ON o.ParentID = h.ID
)
SELECT ...
```
Document: The hierarchy resolution logic and depth handling.

## Output

For each entity, the tracer produces:
1. **Column mapping table** (see mapping-guide.md for structure)
2. **Transform details** for complex SQL logic
3. **Lookup dependencies** with load-order implications
4. **Unmapped columns** with explanations
5. **Data quality notes** (truncation, nulls, encoding)
