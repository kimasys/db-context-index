# 99 — Change Log

**PT-BR:** Histórico de migrations com classificação de risco.

---

## Purpose

Every migration applied to this database must be logged here.

This file serves as:
- migration memory across AI sessions
- risk audit trail
- onboarding reference for new developers and agents
- rollback reference

---

## Template — Migration Entry

```md
## YYYY-MM-DD — migration_name

Risk: LOW | MEDIUM | HIGH
Author: name or AI session reference
Status: applied | rolled back | partial

### Objects affected
- list of tables, functions, policies changed

### RLS impact
- description or "none"

### What changed
- brief description of the migration

### SQL summary
\```sql
-- key SQL statements (summary, not full dump)
\```

### Rollback
\```sql
-- how to reverse this migration if needed
\```

### Notes
- anything unusual, dependencies observed, issues encountered
```

---

## Change Log

---

## 2026-05-08 — initial_schema

Risk: HIGH
Author: initial setup
Status: applied

### Objects affected
- users, profiles, organizations, memberships, projects, tasks, audit_logs (created)
- RLS policies: all (created)
- Helper functions: get_user_org_ids, is_org_member, is_org_admin (created)
- Triggers: on_auth_user_created, set_profiles_updated_at, set_projects_updated_at, audit_memberships (created)

### RLS impact
Full RLS policy suite established for all public schema tables.

### What changed
Initial schema creation. Established authentication domain (users, profiles), organization domain (organizations, memberships), operations domain (projects, tasks), and system domain (audit_logs).

### SQL summary
```sql
-- See schema/initial/ for full migration files
create table profiles (...);
create table organizations (...);
create table memberships (...);
create table projects (...);
create table tasks (...);
create table audit_logs (...);
-- RLS enabled on all tables
-- Helper functions created
-- Triggers created
```

### Rollback
```sql
-- Drop all tables in reverse dependency order
drop table if exists audit_logs;
drop table if exists tasks;
drop table if exists projects;
drop table if exists memberships;
drop table if exists organizations;
drop table if exists profiles;
-- Drop functions and triggers
```

### Notes
This is the base schema. All subsequent migrations build on this foundation. Any change to memberships or RLS helper functions carries HIGH risk.

---

## [NEXT MIGRATION GOES HERE]

---

## Change Log Rules

1. Every migration must be logged before it is considered complete.
2. Risk classification is mandatory — use the criteria in `05-migration-risks.md`.
3. Rollback SQL must be documented even for LOW risk migrations.
4. After a migration, update all affected index files (tables, RLS, functions, relationships).
5. For HIGH risk migrations: note the deployment window and who approved it.
