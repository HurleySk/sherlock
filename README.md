# Sherlock

ETL/data migration analysis skills for Claude Code. Sherlock provides structured analytical frameworks for tracing data lineage, generating mapping documents, and reviewing pipeline changes — with any ETL stack.

Given a codebase with ETL pipelines, SQL transforms, and target system schemas, Sherlock can trace data lineage from source to sink, generate mapping documents with column-level precision, document transform logic, and orchestrate multi-agent pipeline reviews.

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
| lineage-tracer | `/sherlock:lineage-tracer` | Trace data lineage through ETL pipelines from source to sink with column-level precision |
| mapping-guide | `/sherlock:mapping-guide` | Reference guide for correct structure of migration mapping documents |
| pipeline-review | `/sherlock:pipeline-review` | Multi-agent orchestrated review of pipeline, SQL, and migration changes |

## What it does

- **Migration Analyst**: Orchestrates end-to-end migration analysis. Identifies source/target systems, traces data flows, and produces structured mapping documents. Adapts terminology to your ETL stack (ADF, dbt, SSIS, Informatica, Spark, etc.).
- **Lineage Tracer**: Walks through pipeline definitions and SQL transforms to trace data from source tables through staging to target systems, capturing column-level transformations at each hop. Includes tool-specific guidance for ADF, dbt, SSIS, stored procedure-based, and code-based ETL.
- **Mapping Guide**: Defines the canonical structure for mapping documents — entity summaries, column mapping tables, transform details, lookup dependencies, unmapped columns, and data quality notes.
- **Pipeline Review**: Dispatches subagents for deep step-level analysis of pipeline changes, then verifies and consolidates findings. 7-phase orchestration: Scope, Partition, Dispatch, Recurse, Verify, Consolidate, Learn.

## Structure

```
sherlock/
  .claude-plugin/        # Plugin manifest for marketplace
  skills/sherlock/       # Skill definitions (4 skills)
  templates/             # Reusable document templates (column mapping, entity inventory)
  references/            # Domain reference material (ETL patterns)
```
