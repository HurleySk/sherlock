---
name: mapping-guide
description: "Reference guide for correct structure of ETL/data migration mapping documents"
---

# Mapping Guide

## The Golden Rule

A mapping document maps **source to sink**. Every row in a mapping table answers: "Where does this data come from, where does it go, and what happens to it along the way?"

The source is the legacy/origin system. The sink is the target system. Everything in between (staging tables, transform SQL, pipeline activities) is implementation detail that belongs in the Transform column, not as its own mapping layer.

## Correct Mapping Table Structure

| Source Table | Source Column | Source Type | DV Entity | DV Attribute | DV Type | Transform | Lookup Dependencies | Notes |
|---|---|---|---|---|---|---|---|---|
| dbo.Organization | OrgID | int | alm_organization | alm_organizationid | Uniqueidentifier | Lookup via stg_Organization on OrgID | stg_Organization must be loaded first | Legacy FK resolved to DV GUID |
| dbo.Organization | OrgName | varchar(200) | alm_organization | alm_name | Single Line of Text | LTRIM(RTRIM(OrgName)) | None | Whitespace cleanup |
| dbo.Organization | OrgTypeCode | int | alm_organization | alm_organizationtype | Choice | CASE OrgTypeCode WHEN 1 THEN 455780000 WHEN 2 THEN 455780001 END | None | Legacy code to DV option set |

Each row traces ONE column from source to sink. The Transform column documents what happens in between, including which staging table is involved and any SQL logic.

## Section Structure for an Entity Mapping Document

### 1. Entity Summary

| Field | Value |
|---|---|
| Source Table | `dbo.Organization` |
| Target DV Entity | `alm_organization` |
| Pipeline | `onprem_NightlyOrganizationLoad` |
| Alternate Key | `alm_legacyorgid` |
| Upsert Behavior | Match on alternate key, create or update |
| Record Count (approx) | 12,000 |

### 2. Column Mapping Table

The main artifact. See structure above. Every source column appears here, either mapped to a DV attribute or listed in the Unmapped Columns section.

### 3. Transform Details

For complex transforms that cannot fit in a single table cell, expand with specifics:

**OrgTypeCode mapping**: The legacy system uses integer codes 1-5. Dataverse uses option set values 455780000-455780004. Full mapping:
| Legacy Code | DV Option Set Value | Label |
|---|---|---|
| 1 | 455780000 | Federal |
| 2 | 455780001 | State |

**Hierarchy resolution**: Parent org lookups use a CTE to resolve the full tree before joining...

### 4. Lookup Dependencies

What reference data must be loaded before this entity can be migrated.

| Lookup Table | Source | Used By (Column) | Join Logic | Purpose |
|---|---|---|---|---|
| stg_Organization | Legacy dbo.Organization | alm_parentorganizationid | JOIN on ParentOrgID = OrgID | Resolve parent org FK to DV GUID |

### 5. Unmapped Columns

Source columns not migrated to the target system.

| Source Column | Reason |
|---|---|
| LastModifiedDate | System-managed in Dataverse |
| InternalNotes | Not in scope per business decision |

### 6. Data Quality Notes

- OrgName: 3 records exceed 200-char limit, will be truncated
- OrgTypeCode: 47 records have NULL, mapped to default value 455780000
- Phone: Mixed formats, no normalization applied (carried as-is)

## Entity Inventory Structure

For documenting all entities in a migration project:

| Source Table | Target DV Entity | Pipeline | Status | Column Count (Source) | Column Count (Target) | Mapped in Doc |
|---|---|---|---|---|---|---|
| dbo.Organization | alm_organization | onprem_NightlyOrganizationLoad | Complete | 24 | 18 | Yes |
| dbo.Meeting | alm_meeting | onprem_NightlyMeetingLoad | In Progress | 42 | 35 | Partial |

## What NOT to Do

- **Don't map staging to sink.** Staging is the mechanism, not the source. Always start from the legacy/origin system.
- **Don't describe pipeline activities as the mapping.** "Copy Activity 1 copies from on-prem to staging" is implementation, not mapping. The mapping is "dbo.Organization.OrgName goes to alm_organization.alm_name via LTRIM/RTRIM".
- **Don't skip the source system.** Even if you only have access to staging tables, trace back to what populated them.
- **Don't confuse lookup tables with migration targets.** Lookup/reference tables support the migration. Document them separately.

## Lookup Table Documentation

Lookup tables are supporting infrastructure for the migration. Document them in their own section:

| Lookup Table | Source | Used By (Migration Entities) | Join Columns | Purpose |
|---|---|---|---|---|
| stg_SecurityRole | Legacy dbo.SecurityRole | alm_user, alm_team | RoleID | Resolve role assignments to DV GUIDs |
| stg_BusinessUnit | Legacy dbo.BusinessUnit | alm_organization, alm_user | BUID | Resolve BU hierarchy |
