# Common ETL Patterns in Data Migration

A concise reference for recognizing and documenting common patterns encountered in ETL/data migration projects.

## Copy-Transform-Load (Staging Table Pattern)

The most common pattern. Data flows through three hops:

1. **Copy**: Raw data from source system is copied to a staging table in Azure SQL. Minimal transformation — just a faithful copy.
2. **Transform**: A SQL query (embedded in a Copy activity or stored procedure) reads from staging, applies JOINs/CASE/string functions, and produces the output columns.
3. **Load**: The transformed output is upserted into the target system (Dataverse) via Copy activity with alternate key matching.

**Recognition**: Pipeline has two Copy activities in sequence. First has an on-prem source and Azure SQL sink with a `TRUNCATE` preCopyScript. Second has Azure SQL source (with a complex `sqlReaderQuery`) and Dataverse sink.

## Lookup Resolution (FK to GUID)

Legacy systems use integer foreign keys. Target system (Dataverse) uses GUIDs. Resolution happens via JOIN to a reference/staging table that maps legacy IDs to target GUIDs.

```sql
LEFT JOIN stg_Organization org ON src.OrgFK = org.LegacyOrgID
-- Output: org.alm_organizationid AS _alm_organizationid_value
```

**Recognition**: LEFT JOIN in the transform query to a `stg_*` table, outputting a GUID column. The staging lookup table must be loaded before the entity that references it.

**Documentation**: Note the lookup table, join columns, and load-order dependency.

## Option Set Mapping (Legacy Codes to DV Picklist)

Legacy systems store status/type as integers or short strings. Dataverse uses option set values (typically 455780000+).

```sql
CASE src.StatusCode
    WHEN 'A' THEN 455780000
    WHEN 'I' THEN 455780001
    WHEN 'P' THEN 455780002
    ELSE NULL
END AS alm_status
```

**Recognition**: CASE statement in the transform query mapping discrete values to 455780000-range integers.

**Documentation**: Create a mapping table showing legacy value, DV option set value, and human-readable label.

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

Many-to-many relationships in the legacy system are represented as junction tables. In Dataverse, these become N:N relationship associations.

**Flow**:
1. Load both related entities first
2. Query the legacy junction table for pairs of IDs
3. Resolve both IDs to DV GUIDs
4. Use the Dataverse `$ref` API (or `$batch` for bulk) to create associations

**Recognition**: Three-column junction table (JunctionID, Entity1FK, Entity2FK), or a Dataverse N:N relationship being populated.

**Documentation**: Note the relationship name, both related entities, the source of pairs, and batch/parallelism settings.

## Polymorphic Lookups

A single foreign key column in the legacy system can point to different entity types depending on context. In Dataverse, these are polymorphic lookup fields (Customer, Owner, Regarding).

```sql
CASE src.RelatedEntityType
    WHEN 'Account' THEN '/accounts(' + CAST(acct.alm_accountid AS VARCHAR(36)) + ')'
    WHEN 'Contact' THEN '/contacts(' + CAST(cont.alm_contactid AS VARCHAR(36)) + ')'
END AS "alm_regardingid@odata.bind"
```

**Recognition**: CASE statement that constructs different `@odata.bind` URIs based on a type discriminator column.

**Documentation**: Note all possible target entity types, the discriminator column/logic, and the lookup resolution for each type.

## Ownership and Team Assignment Chains

Migrating record ownership involves multiple related entities loaded in strict order:

1. **Business Units**: Establish the organizational hierarchy
2. **Teams**: Created within business units
3. **Users**: Assigned to business units
4. **Security Roles**: Assigned to users/teams
5. **Record Ownership**: Records assigned to users or teams

**Recognition**: Pipeline orchestrator runs child pipelines in a specific sequence. Security model configuration tables control which environments are processed.

**Documentation**: Note the full chain, the configuration tables involved (e.g., `ADF_Security_Model_Config`), and the environment-specific parameters (root BU GUID, service URI).
