# Entity Inventory: {Project Name}

## Overview

| Metric | Value |
|---|---|
| Total Source Tables | {count} |
| Total Target DV Entities | {count} |
| Fully Mapped | {count} |
| In Progress | {count} |
| Not Started | {count} |

## Entity Inventory

| Source Table | Target DV Entity | Pipeline | Status | Column Count (Source) | Column Count (Target) | Mapped in Doc |
|---|---|---|---|---|---|---|
| {dbo.Organization} | {alm_organization} | {onprem_NightlyOrganizationLoad} | {Complete} | {24} | {18} | {Yes} |
| {dbo.Meeting} | {alm_meeting} | {onprem_NightlyMeetingLoad} | {In Progress} | {42} | {35} | {Partial} |
| {dbo.Document} | {alm_document} | {onprem_NightlyDocumentLoad} | {Not Started} | {56} | {0} | {No} |

## Lookup/Reference Tables

| Lookup Table | Source | Used By (Migration Entities) | Join Columns | Purpose |
|---|---|---|---|---|
| {stg_BusinessUnit} | {Legacy dbo.BusinessUnit} | {alm_organization, alm_user} | {BUID} | {Resolve BU hierarchy} |
| {stg_SecurityRole} | {Legacy dbo.SecurityRole} | {alm_user, alm_team} | {RoleID} | {Resolve role assignments} |

## Load Order

Entities must be loaded in dependency order. Parent/reference entities before children.

1. {Reference data (lookup tables)}
2. {Parent entities (organizations, business units)}
3. {Child entities (users, teams)}
4. {Transactional entities (meetings, documents)}
5. {Junction/N:N associations}
