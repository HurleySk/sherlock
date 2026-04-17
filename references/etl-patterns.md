# Common ETL Patterns in Data Migration

A concise reference for recognizing and documenting common patterns encountered in ETL/data migration projects.

## Copy-Transform-Load (Staging Table Pattern)

The most common pattern. Data flows through three hops:

1. **Copy**: Raw data from source system is copied to a staging table in the staging database. Minimal transformation — just a faithful copy.
2. **Transform**: A SQL query (embedded in a pipeline activity or stored procedure) reads from staging, applies JOINs/CASE/string functions, and produces the output columns.
3. **Load**: The transformed output is upserted into the target system via pipeline activity with alternate key matching.

**Recognition**: Pipeline has two activities in sequence. First has a source system connection and staging database sink with a `TRUNCATE` preCopyScript. Second has a staging database source (with a complex transform query) and target system sink.

## Lookup Resolution (FK to Target Key)

Legacy systems use integer foreign keys. Target systems may use different key types (GUIDs, surrogate keys, natural keys). Resolution happens via JOIN to a reference/staging table that maps legacy IDs to target keys.

```sql
LEFT JOIN stg_Organization org ON src.OrgFK = org.LegacyOrgID
-- Output: org.target_org_id AS target_department_id
```

**Recognition**: LEFT JOIN in the transform query to a `stg_*` table, outputting a target key column. The staging lookup table must be loaded before the entity that references it.

**Documentation**: Note the lookup table, join columns, and load-order dependency.

## Code/Enum Mapping (Legacy Codes to Target Values)

Legacy systems store status/type as integers or short strings. Target systems use their own code sets (enums, integer codes, string codes).

```sql
CASE src.StatusCode
    WHEN 'A' THEN 1
    WHEN 'I' THEN 2
    WHEN 'P' THEN 3
    ELSE NULL
END AS target_status
```

**Recognition**: CASE statement mapping discrete source values to target-system-specific codes.

**Documentation**: Create a mapping table showing legacy value, target value, and human-readable label.

## Hierarchy Loading (Self-Referential Tables)

Entities with parent-child relationships (organizations, business units, categories) require special handling. The parent record must exist before the child can reference it.

**Strategies**:
- **Two-pass load**: First pass loads all records without parent references. Second pass updates parent lookups.
- **CTE-based ordering**: A recursive CTE determines load order by tree level.
- **Top-down load**: Load root nodes first, then children in level order.

```sql
WITH Hierarchy AS (
    SELECT ID, ParentID, 0 AS Level FROM Entity WHERE ParentID IS NULL
    UNION ALL
    SELECT e.ID, e.ParentID, h.Level + 1 FROM Entity e JOIN Hierarchy h ON e.ParentID = h.ID
)
```

**Recognition**: Self-referential foreign key (ParentID references same table's ID), recursive CTEs, multi-pass pipeline logic.

**Documentation**: Note the hierarchy depth, root node identification, and whether the load is single-pass or multi-pass.

## Partitioned Loading (ID Range Batching)

Large datasets are split into batches by ID range to avoid timeouts and memory pressure.

```sql
WHERE EntityID BETWEEN @StartID AND @EndID
```

**Recognition**: Pipeline uses ForEach activity with ID range parameters, or stored procedure accepts start/end parameters.

**Documentation**: Note the partition column, batch size, and any ordering requirements.

## N:N Junction Tables

Many-to-many relationships in the legacy system are represented as junction tables. In the target system, these become N:N relationship associations or junction/bridge tables.

**Flow**:
1. Load both related entities first
2. Query the legacy junction table for pairs of IDs
3. Resolve both IDs to target keys
4. Insert associations via target system API or junction table

**Recognition**: Three-column junction table (JunctionID, Entity1FK, Entity2FK), or a target system N:N relationship being populated.

**Documentation**: Note the relationship name, both related entities, the source of pairs, and batch/parallelism settings.

## Polymorphic Lookups

A single foreign key column in the legacy system can point to different entity types depending on context. This is common in CRM systems (Dataverse, Salesforce) as polymorphic lookup fields (Customer, Owner, Regarding).

```sql
CASE src.RelatedEntityType
    WHEN 'Account' THEN '/accounts(' + CAST(acct.target_accountid AS VARCHAR(36)) + ')'
    WHEN 'Contact' THEN '/contacts(' + CAST(cont.target_contactid AS VARCHAR(36)) + ')'
END AS regarding_id_ref
```

**Recognition**: CASE statement that constructs different entity references based on a type discriminator column. The exact syntax depends on the target API (e.g., OData entity references, API resource paths, foreign key with type discriminator column).

**Documentation**: Note all possible target entity types, the discriminator column/logic, and the lookup resolution for each type.

## Dedup Before Joining (Lookup Fan-Out Prevention)

When a transform JOIN resolves a foreign key via a lookup/staging table, duplicate keys in the lookup table cause row fan-out — one source row becomes many output rows. This silently inflates record counts and can create duplicate upserts in the target.

```sql
-- Dedup the lookup table before joining
WITH EmployeeDedup AS (
    SELECT employee_id, legacy_employee_id,
        ROW_NUMBER() OVER(
            PARTITION BY legacy_employee_id
            ORDER BY employee_id
        ) AS rn
    FROM stg_employee
    WHERE legacy_employee_id IS NOT NULL
)
LEFT JOIN EmployeeDedup emp
ON src.EmployeeFK = emp.legacy_employee_id AND emp.rn = 1
```

**Recognition**: LEFT JOIN to a staging/lookup table where the join key is not guaranteed unique. Also: unexplained row count increases after a transform step.

**Detection**: Before writing any lookup JOIN, check for duplicates:
```sql
SELECT join_key_column, COUNT(*) AS cnt
FROM lookup_table
WHERE join_key_column IS NOT NULL
GROUP BY join_key_column
HAVING COUNT(*) > 1
```

**Documentation**: Note which lookup tables were deduped, the partition/order strategy chosen, and whether the duplicates represent real data issues or expected multi-valued lookups.

## Shared Staging Table Isolation

When multiple pipelines or pipeline runs share the same staging table (TRUNCATE + reload), concurrent execution causes data corruption — one run truncates mid-read by another.

**Recognition**: Multiple pipelines reference the same staging table in their `preCopyScript` (TRUNCATE) or sink configuration. Orchestrator pipelines use ForEach or parallel execution patterns.

**Mitigations**:
- **Sequential execution**: Ensure pipelines sharing staging tables run serially (`isSequential: true` in ForEach, or explicit dependency chains)
- **Dedicated staging tables**: Give each pipeline its own staging table (e.g., `stg_employee_nightly` vs `stg_employee_security`)
- **Parameterized table names**: Use pipeline parameters to route each run to a different staging table

**Documentation**: Note which staging tables are shared, which pipelines share them, and what isolation mechanism prevents concurrent corruption.

## Schema Does Not Equal Data Flow

A staging table's DDL (CREATE TABLE) defines what columns *can* hold data — not what columns *actually receive* data. The pipeline's extraction step (query, FetchXML attribute list, column mapping) determines what flows in. Columns not requested by the extraction step exist in the schema but contain NULLs in every row.

**Recognition**: A staging table has columns that a transform JOIN or downstream step depends on, but the extraction step's SELECT/attribute list doesn't include them.

**Why it matters**: Assuming a column has data because it exists in the DDL leads to JOINs against all-NULL columns. The pipeline silently inserts NULLs for unrequested columns without error.

**How to verify**: For any staging column used in a transform or JOIN, trace backward to the extraction step and confirm the column appears in its SELECT list, FetchXML `<attribute>` list, or column mapping. If it doesn't, the column must be added to the extraction step before the downstream logic will work.

**Documentation**: When mapping columns through staging, note both the DDL column and the extraction step that populates it. Flag any staging columns that exist in DDL but are not populated.

## Ownership and Team Assignment Chains

Migrating record ownership involves multiple related entities loaded in strict order:

1. **Business Units**: Establish the organizational hierarchy
2. **Teams**: Created within business units
3. **Users**: Assigned to business units
4. **Security Roles**: Assigned to users/teams
5. **Record Ownership**: Records assigned to users or teams

**Recognition**: Pipeline orchestrator runs child pipelines in a specific sequence. Configuration tables or parameters control which target environments are processed.

**Documentation**: Note the full chain, the configuration tables involved, and the environment-specific parameters (root hierarchy ID, service URI).
