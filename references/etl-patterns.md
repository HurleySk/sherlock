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

## Ownership and Team Assignment Chains

Migrating record ownership involves multiple related entities loaded in strict order:

1. **Business Units**: Establish the organizational hierarchy
2. **Teams**: Created within business units
3. **Users**: Assigned to business units
4. **Security Roles**: Assigned to users/teams
5. **Record Ownership**: Records assigned to users or teams

**Recognition**: Pipeline orchestrator runs child pipelines in a specific sequence. Configuration tables or parameters control which target environments are processed.

**Documentation**: Note the full chain, the configuration tables involved, and the environment-specific parameters (root hierarchy ID, service URI).
