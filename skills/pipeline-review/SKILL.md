---
name: pipeline-review
description: "Multi-agent orchestrated review of ETL pipeline, SQL, and data migration changes. Dispatches subagents for deep step-level analysis, then verifies and consolidates findings. Use after making changes to pipelines, transforms, staging tables, or config."
argument-hint: "[file paths to scope review, or --continue to resume]"
---

# Pipeline Review — Multi-Agent Orchestrator

A generalized reviewer for ETL/data migration infrastructure: pipelines, stored procedures, staging tables, target system mappings, and deployment config. Dispatches subagents for deep per-step analysis, then verifies their work and produces a consolidated review.

## Invocation

- **Slash command**: `/sherlock:pipeline-review` (optional args: file paths to scope the review)
- **No args**: Auto-detects changed files from `git diff` (unstaged + staged)
- **With args**: Scopes review to specified paths and their dependencies
- **`--continue`**: Re-runs only subagents whose `NEED_INFO` items now have answers

## Phase 1: Scope

### 1. Identify what changed

If no explicit paths given, run:
```
git diff --name-only
git diff --cached --name-only
```

Classify each changed file by type. Common classifications:

| Pattern | Type | Trace direction |
|---|---|---|
| Pipeline definitions (JSON, YAML, .dtsx, .py) | Pipeline | Trace sources, sinks, dependency chain |
| SQL scripts / stored procedures | Transform logic | Trace which pipeline calls it, what tables it reads/writes |
| Table DDL / schema files | Schema change | Trace which transforms and pipelines use this table |
| Environment config / parameters | Configuration | Check parameterization coverage across environments |
| dbt models (.sql in models/) | Transform model | Trace upstream refs, downstream dependents |

Adapt these classifications to the project's specific ETL tool and file structure.

### 2. Map dependencies

For each changed file, identify:
- **Upstream**: What feeds data into it (source tables, staging tables, prior transforms)
- **Downstream**: What consumes its output (next pipeline step, target table, dependent model)
- **Siblings**: Other files that handle the same entity/table

### 3. Determine review scope

- If args specify a **single pipeline**: review that pipeline and all its referenced transforms/tables
- If args specify a **single transform** (SP, dbt model, script): review that transform, its input tables, and the pipeline(s) that call it
- If no args (git diff): review all changed files and their immediate dependencies
- If `--continue`: read prior review output, check which `NEED_INFO` items have new data

## Phase 2: Partition

### 1. Parse the pipeline definition

Read the full pipeline definition. Extract all steps/activities with their:
- Name, type, dependencies
- For extraction steps: source connection, query/table, sink table
- For transform steps: SQL, script reference, input/output
- For load steps: target connection, write behavior (upsert/append/merge), key strategy
- For orchestration steps: child pipeline/job references, parameters

### 2. Build dependency DAG

Map all step dependencies. Identify:
- Entry points (steps with no dependencies)
- Terminal steps (nothing depends on them)
- Linear chains vs parallel branches

### 3. Group steps by purpose

Classify each step and group into logical units:

| Group | Step patterns |
|---|---|
| **Source extraction** | Steps that read from source systems and write to staging/landing |
| **Reference data pulls** | Steps that pull lookup/reference data from the target system back to staging |
| **Transforms** | Steps that run SQL, stored procedures, or code to transform staging data |
| **Target loading** | Steps that write transformed data to the target system |
| **Orchestration** | Steps that invoke child pipelines/jobs, control flow (if/foreach/switch) |
| **Logging/Validation** | Steps that log progress or validate data quality |

Steps that don't fit neatly: include them in the group of their closest dependency neighbor.

### 4. Create review directory

```
output/reviews/{YYYY-MM-DD}_{pipeline-slug}/
```

Write `_manifest.md`:
```markdown
# Review Manifest

**Date**: {date}
**Pipeline(s)**: {names}
**Scope**: {git diff summary or explicit paths}
**Groups identified**: {count}

## Step Groups
1. {group name} — {N steps} — subagent: {pending/dispatched/complete}
2. ...
```

## Phase 3: Dispatch

For each step group, dispatch a subagent. **Dispatch groups in parallel** when they have no data dependencies on each other.

### Subagent prompt template

```
Analyze the following pipeline step group.

PIPELINE: {pipeline name}
GROUP: {group description}
OUTPUT_FILE: output/reviews/{review-id}/{NN}-{group-slug}.md

STEPS:
{Definition of just the steps in this group}

FILES_TO_READ:
- Pipeline: {pipeline definition path}
- Transforms: {list SP/script/model paths referenced by steps in this group}
- Schemas: {list table DDL or schema files referenced}

UPSTREAM_CONTEXT:
{What feeds this group — e.g., "Source extraction steps have populated staging tables with raw data."}

DOWNSTREAM_CONTEXT:
{What consumes this group's output — e.g., "Load steps will run after this group completes, reading transformed data for upsert into the target system."}

SPECIFIC_QUESTIONS:
{Any targeted concerns from the orchestrator}

REFERENCE_MATERIAL:
- Use /sherlock:lineage-tracer methodology for tracing data flow through steps
- See ../../references/etl-patterns.md for common ETL pattern recognition

Write your assessment to OUTPUT_FILE with this structure:
- FINDINGS: Issues found (rated HIGH/MED/LOW with file:line citations)
- RISKS: Potential problems that need verification
- CHECKS_PASSED: What you verified is correct
- NEED_INFO: What data you'd need to verify uncertain findings
- RECURSE_INTO: Any child pipelines/jobs that need separate analysis
```

### Dispatch rules

- Maximum **5 subagents in parallel** per pipeline (avoid context thrashing)
- If a pipeline has >5 groups, dispatch in waves: first 5, wait for results, then next batch
- For **scoped reviews** (single transform changed): dispatch only 1-2 subagents covering the changed file and its immediate upstream/downstream steps

## Phase 4: Recurse

After subagents complete:

### 1. Read all assessment files

Read each `output/reviews/{review-id}/{NN}-*.md` file.

### 2. Process RECURSE_INTO references

For each `RECURSE_INTO` in subagent assessments:
- Find the child pipeline/job definition
- If found: partition the child pipeline and dispatch new subagents (repeat Phase 2-3)
- If not found: flag as `NEED_INFO`

### 3. Update manifest

Update `_manifest.md` with subagent completion status and any child pipeline dispatches.

## Phase 5: Verify

The orchestrator does NOT re-do subagent analysis. Apply targeted spot-checks:

### 1. Data assumption verification (CRITICAL)

For any pipeline change that introduces or modifies a JOIN:
- **Do NOT rely on staging/intermediate table data to validate JOIN correctness.** Staging tables are intermediaries with incomplete data.
- Flag any JOIN that links cross-system IDs (e.g., legacy FK to target lookup field) as requiring **first-principles validation**: query the actual source system and the actual target system independently, then cross-check using identifying fields (names, emails, codes) — not just key overlap.
- **Do NOT assume overlapping IDs mean a relationship.** Different source tables can have overlapping numeric ID ranges that refer to completely different records. A high match count does not prove correctness. Always verify by comparing an independent identifying field between the joined records.
- **Commingled ID formats**: A single target field can contain values from multiple source pipelines with different ID formats (e.g., prefixed IDs from one pipeline, raw numeric from another). Any normalization that strips prefixes or reformats values to create more matches can produce thousands of false positives if the formats represent different ID systems. Investigate each format's provenance separately before normalizing.
- If the review can't validate the JOIN from code alone, add a NEED_INFO item specifying: (1) the source system query needed with identifying fields, (2) the target system query needed with identifying fields, (3) the cross-check criteria (e.g., "matching key values should produce matching person names").

### 2. Claim verification

For each FINDING or RISK rated HIGH:
- Read the file and line the subagent cited
- Verify the code actually says what the subagent claims
- If the subagent misread the code or cited wrong line numbers, note the discrepancy

### 3. Null semantics and error handling

For each load/upsert step, verify two critical settings:

**Null handling behavior**: Most target systems have a setting controlling whether NULL source values overwrite existing target values or are skipped. (In ADF: `ignoreNullValues`; in dbt: `merge` strategy with column lists; in SSIS: NULL handling in destination adapters.) A LEFT JOIN that doesn't match produces NULL — if the target system writes NULLs, this **clears existing data**. Before deploying:
1. Check how many target records currently have the field populated
2. Check how many source records will match the JOIN
3. If matches < currently populated, the unmatched records will lose data

**Silent error swallowing**: Many ETL tools have settings to skip incompatible/failed rows silently (ADF: `enableSkipIncompatibleRow`; SSIS: error output routing to /dev/null; dbt: `on_schema_change: 'ignore'`). Flag any step with silent error handling enabled. Ask: "What failure modes are expected here?" If none, the setting may be masking real data issues. The pipeline reports success while records are silently dropped.

### 4. Cross-group consistency

Look for contradictions or gaps between group assessments:
- If one group says "data looks correct" but another group's transform assumes different column names — naming mismatch
- If a transform group says it reads from a table that another group shows is populated AFTER the transform runs — execution order conflict
- If extraction says "staging table has column X" but load group references column Y — schema drift
- If a lookup JOIN depends on a staging column, verify the extraction step actually populates that column (see "Schema Does Not Equal Data Flow" in `../../references/etl-patterns.md`)

### 5. Verify against live state

Compare changed files against deployed/live state where accessible:
- Pipeline definitions: repo version vs deployed version
- Transform SQL: repo version vs database version
- Schemas: expected columns vs actual columns

### 6. Deployment safety

- If pipeline references environment-specific configuration, verify coverage across all target environments
- Check that parameterization handles environment differences (connection strings, credentials, paths)
- Verify the deployment/promotion path is documented

## Phase 6: Consolidate

### 1. Merge findings

Collect all findings across subagent assessments and orchestrator verification. Deduplicate.

### 2. Consolidate NEED_INFO items

Collect all `NEED_INFO` across subagents:
- Deduplicate (multiple subagents may need the same data)
- For each unique info need, describe what query or check would resolve it
- Present to user: what info is needed and why

### 3. Write consolidated review

Write `output/reviews/{review-id}/_consolidated-review.md`:

```markdown
# Pipeline Review: {summary}

**Date**: {date}
**Scope**: {what was reviewed}
**Subagents dispatched**: {count}

## Critical Findings
- [FINDING] **HIGH** — {description} — `{file}:{line}` — assessed by: {group} — verified: {yes/no/discrepancy}

## Risks
- [RISK] **HIGH** — {description} — `{file}:{line}` — assessed by: {group} — pending: {NEED_INFO ref if applicable}
- [RISK] **MED** — ...

## Checks Passed
{Aggregated list from all subagents + orchestrator checks}

## Deployment Safety
{Results from deployment safety checks}

## Orchestrator Verification Notes
- {Any discrepancies found in subagent claims}
- {Cross-group consistency issues}

## Information Gaps
| # | What's needed | Why | Affects |
|---|---|---|---|
| 1 | {description} | {reason} | {which findings} |

## Individual Assessments
- [{group name}]({NN}-{slug}.md) — {summary line}
- ...
```

## Phase 7: Learn

After completing the review, if any novel pattern or pitfall was discovered:

1. Read or create `knowledge.md` alongside the pipeline-review skill file, or in the project's `.claude/skills/` directory if the skill is used as a local project skill
2. Draft a new checklist item with:
   - What to check
   - Why it matters (the failure mode)
   - How to check it (specific things to look for in code)
3. Present the proposed addition to the user
4. If approved, append it to `knowledge.md`

## References

- `/sherlock:lineage-tracer` — Subagents can use lineage tracing methodology for deep step analysis
- `/sherlock:mapping-guide` — Reference for mapping document structure if review identifies mapping concerns
- `../../references/etl-patterns.md` — Common ETL patterns for recognition during partition and verify phases

## Notes

- **Subagent context**: Each subagent is fresh — no memory of other subagents' work. The orchestrator is the only entity with the full picture. Cross-group verification in Phase 5 is essential.
- **Large pipelines**: Pipelines with 25+ steps will generate multiple subagents. Dispatch in waves of 5 max.
- **Scoped reviews**: When only 1-2 files changed, don't analyze the entire pipeline — scope to the changed step group + immediate neighbors.
- **Re-invocation with --continue**: Reads prior assessments, checks for new data, re-dispatches only subagents whose `NEED_INFO` items are now answerable.
