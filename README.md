# Sherlock

Umbrella plugin for domain-specific analyst skills for Claude Code. Sherlock provides structured analytical frameworks that help Claude Code reason about complex domains — starting with ETL/data migration.

## Install

```
claude skill install HurleySk/sherlock
```

## Skills

| Skill | Command | Description |
|---|---|---|
| migration-analyst | `/sherlock:migration-analyst` | ETL/data migration analysis — mapping docs, lineage tracing, transform documentation |
| mapping-guide | `/sherlock:mapping-guide` | Reference guide for correct structure of migration mapping documents |
| lineage-tracer | `/sherlock:lineage-tracer` | Trace data lineage through ETL pipelines from source to sink with column-level precision |

## Standalone vs Sisyphus

Sherlock works standalone for direct analysis and document generation. It is also Sisyphus-aware: the `templates/sisyphus-spec.json` template generates spec files compatible with the Sisyphus document lifecycle framework, so mapping documents can be authored with structured criteria and review workflows.

## Structure

```
sherlock/
  skills/sherlock/       # Skill definitions
  templates/             # Reusable document templates (markdown + Sisyphus specs)
  references/            # Domain reference material
```
