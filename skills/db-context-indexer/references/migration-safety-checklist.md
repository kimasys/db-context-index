# Migration Safety Checklist

**PT-BR:** Checklist de segurança para migrations SQL com PostgreSQL e Supabase.

---

## Pre-Migration (Before Writing Any SQL)

### Context verification

- [ ] Read `00-overview.md` — confirmed affected domain
- [ ] Read relevant section of `01-tables.md` — confirmed tables involved
- [ ] Checked `02-rls-policies.md` — confirmed RLS impact
- [ ] Checked `03-functions-triggers.md` — confirmed function/trigger impact
- [ ] Checked `04-relationships.md` — confirmed cascade and dependency impact
- [ ] Reviewed `05-migration-risks.md` — identified known fragile areas
- [ ] Risk classified: LOW / MEDIUM / HIGH

### Environment

- [ ] Working against staging, not production
- [ ] Staging data volume is representative of production (order of magnitude)
- [ ] Backup exists before executing

### For HIGH risk migrations

- [ ] Migration reviewed by a second human or AI pass
- [ ] Deployment window is off-peak
- [ ] Rollback plan is written and tested
- [ ] Stakeholder or team lead aware

---

## SQL Writing Checklist

### Additive changes (LOW risk)

- [ ] New columns are nullable OR have a DEFAULT value
- [ ] New indexes use `CONCURRENTLY` to avoid table locks
- [ ] New tables immediately have RLS enabled
- [ ] New RLS policies are consistent with existing policy patterns

```sql
-- Safe: new nullable column
alter table tasks add column priority text;

-- Safe: new index (non-blocking)
create index concurrently idx_tasks_priority on tasks (priority);

-- Safe: new table with immediate RLS
create table attachments (...);
alter table attachments enable row level security;
```

---

### Column changes (MEDIUM–HIGH risk)

- [ ] Column rename: all references updated (functions, triggers, policies, application code)
- [ ] Type change: conversion is safe for all existing data
- [ ] NOT NULL addition: all existing rows have non-null values OR a DEFAULT is set first
- [ ] Large table changes: use concurrent or phased approach to avoid long locks

```sql
-- Safe pattern: add NOT NULL constraint to existing column
-- Step 1: set default
alter table tasks alter column priority set default 'normal';
-- Step 2: backfill nulls
update tasks set priority = 'normal' where priority is null;
-- Step 3: add not null
alter table tasks alter column priority set not null;
```

---

### Destructive changes (HIGH risk)

- [ ] Column drop: confirmed no application code, function, policy, or trigger references it
- [ ] Table drop: confirmed no FK references, no dependent views, no application code
- [ ] Soft-delete alternative considered before hard delete

```sql
-- Before dropping: check for FK dependencies
select conname, conrelid::regclass
from pg_constraint
where confrelid = 'table_name'::regclass;
```

---

### RLS changes (HIGH risk)

- [ ] Existing policies re-read and understood before modification
- [ ] Change does not create a gap (period with no SELECT policy = data invisible to all)
- [ ] Change does not over-grant (period with permissive policy = data exposed)
- [ ] Helper functions tested if modified
- [ ] Tested from client SDK (not SQL Editor — SQL Editor bypasses RLS)
- [ ] Tested as: anonymous role, authenticated user, admin user, service_role

```sql
-- Test as a specific role
set role authenticated;
set request.jwt.claims to '{"sub": "user-uuid", "role": "authenticated"}';
select * from projects;
reset role;
```

---

### Foreign key changes (MEDIUM–HIGH risk)

- [ ] New FK: no orphaned rows exist in the child table
- [ ] Dropped FK: cascade behavior is intentional and documented
- [ ] Changed cascade rule: impact on deletion workflows understood

```sql
-- Check for orphaned rows before adding FK
select count(*) from tasks
where project_id not in (select id from projects);
-- must return 0 before adding FK
```

---

## Post-Migration

- [ ] Migration applied successfully in staging
- [ ] Affected application flows tested end-to-end
- [ ] RLS policies validated from client SDK if changed
- [ ] Performance verified: no new slow queries (check EXPLAIN ANALYZE)
- [ ] `99-change-log.md` updated with full entry
- [ ] All affected index files updated:
  - [ ] `01-tables.md` if tables changed
  - [ ] `02-rls-policies.md` if policies changed
  - [ ] `03-functions-triggers.md` if functions or triggers changed
  - [ ] `04-relationships.md` if FK relationships changed
- [ ] Rollback SQL documented in change log

---

## PostgreSQL Lock Reference

| Operation | Lock Level | Risk |
|---|---|---|
| `ALTER TABLE ADD COLUMN` (nullable) | ACCESS EXCLUSIVE (brief) | LOW |
| `ALTER TABLE ADD COLUMN NOT NULL` (no default) | ACCESS EXCLUSIVE (long) | HIGH |
| `CREATE INDEX` | SHARE | MEDIUM — blocks writes |
| `CREATE INDEX CONCURRENTLY` | None | LOW |
| `DROP INDEX` | ACCESS EXCLUSIVE (brief) | LOW |
| `ALTER TABLE ADD CONSTRAINT` | ACCESS EXCLUSIVE | HIGH |
| `UPDATE` (backfill) | ROW EXCLUSIVE | MEDIUM — row-level locks |

**Rule:** For production tables with >100k rows, use CONCURRENTLY for indexes and phased approaches for constraints.
