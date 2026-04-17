---
name: mapping-guide
description: "Reference guide for correct structure of ETL/data migration mapping documents"
---

# Mapping Guide

## The Golden Rule

A mapping document maps **source to sink**. Every row in a mapping table answers: "Where does this data come from, where does it go, and what happens to it along the way?"

The source is the legacy/origin system. The sink is the target system. Everything in between (staging tables, transform SQL, pipeline activities) is implementation detail that belongs in the Transform column, not as its own mapping layer.

## Correct Mapping Table Structure

| Source Table | Source Column | Source Type | Target Table | Target Column | Target Type | Transform | Lookup Dependencies | Notes |
|---|---|---|---|---|---|---|---|---|
| dbo.Organization | OrgID | int | dim_organization | organization_id | bigint | Lookup via ref_org on OrgID | ref_org must be loaded first | Legacy FK resolved to target surrogate key |
| dbo.Organization | OrgName | varchar(200) | dim_organization | org_name | varchar(200) | LTRIM(RTRIM(OrgName)) | None | Whitespace cleanup |
| dbo.Organization | OrgTypeCode | int | dim_organization | org_type | int | CASE OrgTypeCode WHEN 1 THEN 100 WHEN 2 THEN 200 END | None | Legacy code to target enum |

Each row traces ONE column from source to sink. The Transform column documents what happens in between, including which staging table is involved and any SQL logic.

**Target Type column**: Use whatever type system the target uses — SQL types (varchar, int, bigint), Snowflake types (VARIANT, TIMESTAMP_NTZ), CRM field types (Single Line of Text, Lookup, Choice), warehouse types (STRING, INT64), etc.

## Section Structure for an Entity Mapping Document

### 1. Entity Summary

| Field | Value |
|---|---|
| Source Table | `dbo.Organization` |
| Target Table/Entity | `dim_organization` |
| Pipeline/Job | `load_organizations` |
| Key Strategy | Match on `legacy_org_id`, create or update |
| Record Count (approx) | 12,000 |

### 2. Column Mapping Table

The main artifact. See structure above. Every source column appears here, either mapped to a target column or listed in the Unmapped Columns section.

### 3. Transform Details

For complex transforms that cannot fit in a single table cell, expand with specifics:

**OrgTypeCode mapping**: The legacy system uses integer codes 1-5. The target system uses a different code set. Full mapping:
| Legacy Code | Target Value | Label |
|---|---|---|
| 1 | 100 | Federal |
| 2 | 200 | State |

**Hierarchy resolution**: Parent org lookups use a CTE to resolve the full tree before joining...

### 4. Lookup Dependencies

What reference data must be loaded before this entity can be migrated.

| Lookup Table | Source | Used By (Column) | Join Logic | Purpose |
|---|---|---|---|---|
| ref_org | Legacy dbo.Organization | parent_org_id | JOIN on ParentOrgID = OrgID | Resolve parent org FK to target key |

### 5. Unmapped Columns

Source columns not migrated to the target system.

| Source Column | Reason |
|---|---|
| LastModifiedDate | System-managed in target |
| InternalNotes | Not in scope per business decision |

### 6. Data Quality Notes

- OrgName: 3 records exceed 200-char limit, will be truncated
- OrgTypeCode: 47 records have NULL, mapped to default value
- Phone: Mixed formats, no normalization applied (carried as-is)

## Entity Inventory Structure

For documenting all entities in a migration project:

| Source Table | Target Table/Entity | Pipeline/Job | Status | Column Count (Source) | Column Count (Target) | Mapped in Doc |
|---|---|---|---|---|---|---|
| dbo.Organization | dim_organization | load_organizations | Complete | 24 | 18 | Yes |
| dbo.Meeting | dim_meeting | load_meetings | In Progress | 42 | 35 | Partial |

## What NOT to Do

- **Don't map staging to sink.** Staging is the mechanism, not the source. Always start from the legacy/origin system.
- **Don't describe pipeline activities as the mapping.** "Step 1 copies from source to staging" is implementation, not mapping. The mapping is "dbo.Organization.OrgName goes to dim_organization.org_name via LTRIM/RTRIM".
- **Don't skip the source system.** Even if you only have access to staging tables, trace back to what populated them.
- **Don't confuse lookup tables with migration targets.** Lookup/reference tables support the migration. Document them separately.

## Lookup Table Documentation

Lookup tables are supporting infrastructure for the migration. Document them in their own section:

| Lookup Table | Source | Used By (Migration Entities) | Join Columns | Purpose |
|---|---|---|---|---|
| ref_security_role | Legacy dbo.SecurityRole | dim_user, dim_team | RoleID | Resolve role assignments to target keys |
| ref_business_unit | Legacy dbo.BusinessUnit | dim_organization, dim_user | BUID | Resolve BU hierarchy |
