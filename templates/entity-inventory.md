# Entity Inventory: {Project Name}

## Overview

| Metric | Value |
|---|---|
| Total Source Tables | {count} |
| Total Target Tables/Entities | {count} |
| Fully Mapped | {count} |
| In Progress | {count} |
| Not Started | {count} |

## Entity Inventory

| Source Table | Target Table/Entity | Pipeline/Job | Status | Column Count (Source) | Column Count (Target) | Mapped in Doc |
|---|---|---|---|---|---|---|
| {dbo.Organization} | {dim_organization} | {load_organizations} | {Complete} | {24} | {18} | {Yes} |
| {dbo.Meeting} | {dim_meeting} | {load_meetings} | {In Progress} | {42} | {35} | {Partial} |
| {dbo.Document} | {dim_document} | {load_documents} | {Not Started} | {56} | {0} | {No} |

## Lookup/Reference Tables

| Lookup Table | Source | Used By (Migration Entities) | Join Columns | Purpose |
|---|---|---|---|---|
| {ref_business_unit} | {Legacy dbo.BusinessUnit} | {dim_organization, dim_user} | {BUID} | {Resolve BU hierarchy} |
| {ref_security_role} | {Legacy dbo.SecurityRole} | {dim_user, dim_team} | {RoleID} | {Resolve role assignments} |

## Load Order

Entities must be loaded in dependency order. Parent/reference entities before children.

1. {Reference data (lookup tables)}
2. {Parent entities (organizations, business units)}
3. {Child entities (users, teams)}
4. {Transactional entities (meetings, documents)}
5. {Junction/N:N associations}
