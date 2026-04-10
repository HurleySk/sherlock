---
name: migration-analyst
description: "ETL/data migration analysis — mapping docs, lineage tracing, transform documentation. Use when analyzing data flows from source to sink systems."
---

# Migration Analyst

Invoked as `/sherlock:migration-analyst`.

Analyze data migration artifacts (pipelines, stored procedures, staging tables, schema definitions) and produce or guide production of mapping documents.

## Key Principle

**Mapping = source to sink.** The subject of a mapping document is the journey from the legacy/source system to the target system. Staging tables, transform SQL, and intermediary pipeline steps document the "how", not the "what". Always frame analysis from the perspective of: where does data originate, and where does it end up?

## Workflow

When invoked, follow this sequence:

### 1. Identify Migration Scope

Determine which entities are in scope. Ask or infer:
- Source system (legacy SQL Server tables, flat files, etc.)
- Target system (Dataverse entities, Azure SQL, etc.)
- Which entities/tables are being migrated
- Are there existing mapping documents to validate, or is this a fresh analysis?

### 2. Trace Data Lineage

For each entity in scope, trace the full path from source to sink. Follow the guidance in `lineage-tracer.md`:
- Identify the pipeline or SP responsible for the migration
- Walk through each hop (source, staging, sink)
- Extract column-level mappings with transform details
- Note lookup dependencies and reference data requirements

### 3. Produce or Validate Mapping Artifacts

Generate mapping documents following the structure defined in `mapping-guide.md`:
- Entity summary (source table, target entity, pipeline, alternate key)
- Column mapping table (every row = one column's journey)
- Transform details for complex logic
- Lookup dependency documentation
- Unmapped columns with explanations
- Data quality notes

## Standalone Mode

When working directly (no Sisyphus), produce markdown mapping documents with the correct structure. Use the templates in `templates/` as starting points:
- `templates/column-mapping.md` for per-entity column mapping sections
- `templates/entity-inventory.md` for the overall entity inventory

## Sisyphus Mode

When generating Sisyphus-compatible specs, use `templates/sisyphus-spec.json` as the base. This ensures:
- Correct section structure (entity summary + column mapping per entity)
- Pre-defined criteria that enforce mapping quality
- Gather sources configured for all three data layers

## References

- `mapping-guide.md` — What a mapping document should contain and how to structure it
- `lineage-tracer.md` — How to trace data flow through pipelines and SPs
- `references/etl-patterns.md` — Common ETL patterns for recognition during analysis
