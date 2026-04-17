---
name: migration-analyst
description: "ETL/data migration analysis — mapping docs, lineage tracing, transform documentation. Use when analyzing data flows from source to sink systems."
---

# Migration Analyst

Invoked as `/sherlock:migration-analyst`.

Analyze data migration artifacts (pipelines, stored procedures, staging tables, schema definitions) and produce or guide production of mapping documents.

## Getting Started

Before tracing any lineage, establish the migration context. Ask the user:

1. **ETL tool/stack**: What moves the data? (ADF, dbt, SSIS, Informatica, Talend, Spark, custom scripts, raw SQL)
2. **Source system**: Where does data originate? (SQL Server, Oracle, flat files, APIs, legacy CRM)
3. **Target system**: Where does data land? (data warehouse, CRM, data lake, Snowflake, BigQuery, Dataverse)
4. **Hop count**: How many intermediate layers exist? (direct source-to-target, or source → staging → target, or more)
5. **Existing artifacts**: Are there mapping docs to validate, or is this a fresh analysis?

Adapt terminology throughout the analysis to match the user's stack. Use their tool names, their column type names, their pipeline terminology.

## Key Principle

**Mapping = source to sink.** The subject of a mapping document is the journey from the legacy/source system to the target system. Staging tables, transform SQL, and intermediary pipeline steps document the "how", not the "what". Always frame analysis from the perspective of: where does data originate, and where does it end up?

## Workflow

When invoked, follow this sequence:

### 1. Identify Migration Scope

Determine which entities are in scope:
- Source system tables, files, or API endpoints
- Target system tables, entities, or collections
- Which entities/tables are being migrated
- Whether to validate existing mappings or produce new ones (from Getting Started context)

### 2. Trace Data Lineage

For each entity in scope, trace the full path from source to sink. Follow the guidance in the lineage-tracer skill (`/sherlock:lineage-tracer`):
- Identify the pipeline, job, or process responsible for the migration
- Walk through each hop (source → [staging] → [transform] → sink)
- Extract column-level mappings with transform details
- Note lookup dependencies and reference data requirements

### 3. Produce or Validate Mapping Artifacts

Generate mapping documents following the structure defined in the mapping-guide skill (`/sherlock:mapping-guide`):
- Entity summary (source table, target table/entity, pipeline, key strategy)
- Column mapping table (every row = one column's journey)
- Transform details for complex logic
- Lookup dependency documentation
- Unmapped columns with explanations
- Data quality notes

## Output Modes

### Standalone Mode

Produce markdown mapping documents with the correct structure. Use the templates in `templates/` as starting points:
- `templates/column-mapping.md` for per-entity column mapping sections
- `templates/entity-inventory.md` for the overall entity inventory

### Integrated Mode

When working within a larger migration project that has its own orchestration or spec system, adapt the output format to match the project's conventions while preserving the mapping structure defined in the mapping-guide skill.

## References

- `/sherlock:mapping-guide` — What a mapping document should contain and how to structure it
- `/sherlock:lineage-tracer` — How to trace data flow through pipelines and transforms
- `../../references/etl-patterns.md` — Common ETL patterns for recognition during analysis
