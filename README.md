# Sherlock

Domain analyst skills for Claude Code. Sherlock provides structured analytical frameworks that help Claude Code reason about complex domains — starting with ETL/data migration.

Given a codebase with ADF pipelines, SQL stored procedures, and Dataverse schemas, Sherlock can trace data lineage from source to sink, generate mapping documents with column-level precision, and document transform logic.

## Install

### Via HurleySk marketplace (recommended)

```bash
claude plugin marketplace add HurleySk/claude-plugins-marketplace
claude plugin install sherlock
```

### Direct install

```bash
claude skill install HurleySk/sherlock
```

## Skills

| Skill | Command | Description |
|---|---|---|
| migration-analyst | `/sherlock:migration-analyst` | ETL/data migration analysis — mapping docs, lineage tracing, transform documentation |
| mapping-guide | `/sherlock:mapping-guide` | Reference guide for correct structure of migration mapping documents |
| lineage-tracer | `/sherlock:lineage-tracer` | Trace data lineage through ETL pipelines from source to sink with column-level precision |

## What it does

- **Migration Analyst**: Orchestrates end-to-end migration analysis. Identifies source/target systems, traces data flows, and produces structured mapping documents.
- **Mapping Guide**: Defines the canonical structure for mapping documents — entity summaries, column mapping tables, transform details, lookup dependencies, unmapped columns, and data quality notes.
- **Lineage Tracer**: Walks through ADF pipeline JSON and SQL stored procedures to trace data from source tables through staging to Dataverse entities, capturing column-level transformations at each hop.

## Standalone vs Sisyphus

Sherlock works standalone for direct analysis and document generation. It is also Sisyphus-aware: the `templates/sisyphus-spec.json` template generates spec files compatible with the [Sisyphus](https://github.com/HurleySk/sisyphus) document lifecycle framework, so mapping documents can be authored with structured criteria and review workflows.

## Structure

```
sherlock/
  .claude-plugin/        # Plugin manifest for marketplace
  skills/sherlock/       # Skill definitions
  templates/             # Reusable document templates (markdown + Sisyphus specs)
  references/            # Domain reference material
```
