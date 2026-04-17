---
name: lineage-tracer
description: "How to trace data lineage through ETL pipelines from source to sink with column-level precision"
---

# Lineage Tracer

## The N-Hop Model

Every migrated column passes through one or more hops on its way from source to sink:

1. **Source** (origin system table, file, or API) — extracted by a pipeline step or script to...
2. **Staging** (intermediate table or landing zone) — transformed via SQL, code, or pipeline logic to...
3. **Sink** (target system table, entity, or collection) — loaded via upsert, merge, append, or replace

Most migrations have 2-4 hops. Some skip staging (direct source-to-sink, 2 hops). Others have multiple intermediate layers (raw → staging → curated → target, 4 hops). Identify how many hops exist before tracing.

The goal of lineage tracing is to follow each column through all hops and document what happens at each step.

## How to Trace: General Framework

Regardless of ETL tool, the process is the same:

### Step 1: Find the Pipeline or Process

Locate the pipeline definition, job config, or script responsible for the migration. Look in your project's pipeline directory for the entity name.

### Step 2: Identify Extraction/Load Steps

Each step in the pipeline has an input (source) and output (sink). A typical migration pipeline has at least two steps:
- **Step A**: Source system to staging/landing zone
- **Step B**: Staging/landing zone to target system

Some pipelines combine these or add intermediate steps. Read the step dependencies to understand execution order.

### Step 3: Trace Each Hop

For each hop, extract:
- **Input**: What table/file/API is read? Which columns?
- **Transform**: What SQL, code, or mapping logic is applied?
- **Output**: What table/entity is written? Which columns?

### Step 4: Document the Column Journey

For each column, record:
- Source table and column name
- Every transformation applied (JOINs, CASE, string functions, type conversions)
- Target table and column name
- Any lookup dependencies (reference tables that must be loaded first)

## Tool-Specific Guidance

### Azure Data Factory (ADF)

ADF pipelines are JSON definitions with Copy activities, stored procedure activities, and orchestration activities.

**Finding the pipeline**: Look in your pipeline directory for JSON files. Pipeline names typically follow patterns like `Nightly{Entity}Load`, `PL_{Entity}_Migration`, or `Load_{Entity}`.

**Tracing Copy activities**: Each Copy activity has a `source` and `sink` section. Key fields:
- `sqlReaderQuery` or table reference — the source columns being read
- `preCopyScript` (often `TRUNCATE TABLE stg_*`) — confirms the staging table name
- Sink configuration — the target entity/table, write behavior (upsert/append), key strategy

**Tracing transforms**: The `sqlReaderQuery` in the second Copy activity typically contains the core transformation SQL — JOINs, CASE statements, string functions, CTEs.

### dbt (Data Build Tool)

dbt models are SQL SELECT statements that define transformations.

**Finding the model**: Look in `models/` for `.sql` files named after the target table.

**Tracing transforms**: The SQL in the model IS the transform. Source references use `{{ source('schema', 'table') }}` or `{{ ref('other_model') }}`. Follow `ref()` calls to trace upstream dependencies.

**Lineage**: dbt generates lineage graphs (`dbt docs generate`). Use these to identify upstream/downstream dependencies.

### SSIS (SQL Server Integration Services)

SSIS packages are `.dtsx` XML files with data flow tasks.

**Finding the package**: Look in the SSIS project for `.dtsx` files. Data flows contain source, transform, and destination components.

**Tracing data flows**: Each data flow task has source adapters (OLE DB Source, Flat File Source), transformations (Derived Column, Lookup, Conditional Split), and destination adapters.

### Stored Procedure-Based ETL

For SP-based migrations:

1. **Find the SP** in your database project or schema export directory
2. **Read the SELECT statement**: Source columns are in the FROM/JOIN clauses, output columns are the SELECT aliases
3. **Follow the same hop logic**: The SP embodies the transform step, so trace backward to find the source and forward to find the sink

### Code-Based ETL (Python/Spark/Custom Scripts)

For programmatic pipelines:

1. **Find the script/module** responsible for the entity migration
2. **Trace the read**: What DataFrame, query, or file read brings in source data?
3. **Trace the transforms**: Column renames, type casts, joins, filters, aggregations
4. **Trace the write**: What table/file/API receives the output?

## Common Transform Patterns

These patterns appear across all ETL tools:

### CASE Statements (Code/Enum Mapping)
```sql
CASE src.StatusCode
    WHEN 'A' THEN 1  -- Active
    WHEN 'I' THEN 2  -- Inactive
    ELSE NULL
END AS target_status
```
Document: Legacy code, target value, and label.

### LEFT JOIN (Lookup Resolution)
```sql
LEFT JOIN ref_department dept ON src.DeptFK = dept.LegacyDeptID
-- Output: dept.target_department_id AS parent_department_id
```
Document: Which lookup table, join columns, and which target field it resolves to.

### String Cleanup
```sql
LTRIM(RTRIM(REPLACE(src.Name, CHAR(9), '')))
```
Document: What cleanup is applied and why.

### CTE Chains (Hierarchy Resolution)
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
2. **Transform details** for complex SQL/code logic
3. **Lookup dependencies** with load-order implications
4. **Unmapped columns** with explanations
5. **Data quality notes** (truncation, nulls, encoding)
