# 05 — Migration Risks

**PT-BR:** Mapa de riscos e área de alta periculosidade para migrations.

---

## Risk Classification

| Level | Definition |
|---|---|
| LOW | Additive, reversible, isolated. No impact on existing data or policies. |
| MEDIUM | Structural change with bounded impact. Requires testing but low blast radius. |
| HIGH | Irreversible, cross-domain, or auth/RLS-critical. Requires review and staging validation. |

---

## HIGH Risk Areas — Do Not Touch Without a Plan

### 1. memberships table

Any structural change to `memberships` cascades risk to every RLS policy in the system.

**Why:** Most RLS helper functions (`is_org_member`, `get_user_org_ids`) query this table. Schema changes can silently break all tenant isolation.

**Before touching:**
- Audit all functions in `02-rls-policies.md` that reference `memberships`
- Test all policies in staging with realistic tenant data
- Plan a rollback migration

---

### 2. RLS helper functions

Changes to `get_user_org_ids()`, `is_org_member()`, `is_org_admin()` affect every table they protect.

**Why:** These functions are the core of tenant isolation. A bug here leaks data across org boundaries.

**Before touching:**
- List every policy that calls the function
- Test SELECT, INSERT, UPDATE, DELETE as different roles (authenticated, anon, service_role)
- Never deploy during peak traffic hours

---

### 3. auth.users integration

The trigger `on_auth_user_created` and the `handle_new_user()` function are the bridge between Supabase Auth and your schema.

**Why:** Failures here result in users who can authenticate but have no profile — a hard-to-debug broken state.

**Before touching:**
- Test new user registration flow end-to-end in staging
- Verify the trigger still fires after any auth.users schema changes (Supabase updates may affect this)

---

### 4. Destructive column changes

Column renames, type changes, and NOT NULL additions on large tables.

**Why:** PostgreSQL rewrites the entire table for some type changes, causing long locks. NOT NULL additions without a DEFAULT value fail if any row has NULL.

**Patterns to use:**

```sql
-- Safe: add nullable column first, backfill, then add constraint
alter table tasks add column priority text;
update tasks set priority = 'normal' where priority is null;
alter table tasks alter column priority set not null;
alter table tasks alter column priority set default 'normal';
```

**Never:**
```sql
-- Unsafe: immediate NOT NULL with no default on a large table
alter table tasks add column priority text not null;
```

---

### 5. CASCADE DELETE on production data

`CASCADE DELETE` rules will silently delete downstream rows when a parent is deleted.

**Risk scenarios:**
- Deleting an organization cascades into memberships (all users lose access)
- Deleting a project cascades into tasks (all tasks permanently deleted)

**Best practice:** Prefer soft deletes (`deleted_at timestamp`) over hard deletes on critical entities. Use CASCADE only where it is explicitly desired and documented.

---

## MEDIUM Risk Areas

### Constraint changes

Adding a UNIQUE constraint requires a full table scan and may lock the table.

```sql
-- Safe: create unique index concurrently first, then add constraint
create unique index concurrently idx_profiles_email_unique on profiles (email);
alter table profiles add constraint profiles_email_unique unique using index idx_profiles_email_unique;
```

### Trigger modifications

Adding or removing triggers affects write performance and audit coverage. Removing an audit trigger from `memberships` would create gaps in the security log.

### Index changes

Dropping indexes on large tables is instant but may cause immediate query degradation. Always check which queries use the index before dropping.

---

## LOW Risk Patterns (safe to apply)

- Adding nullable columns with no constraints
- Adding new indexes (use `CONCURRENTLY` to avoid locks)
- Creating new tables (if RLS is added immediately)
- Adding optional FK references
- Creating new RLS policies on existing tables (does not affect existing policies)

---

## Pre-Migration Checklist

Before any migration, an AI agent must confirm:

- [ ] Index files read: 00-overview, relevant domain sections
- [ ] Objects affected: listed and verified in 01-tables
- [ ] RLS impact: inspected in 02-rls-policies
- [ ] Function/trigger impact: inspected in 03-functions-triggers
- [ ] Relationship cascade impact: verified in 04-relationships
- [ ] Risk classified: LOW / MEDIUM / HIGH
- [ ] Rollback plan documented
- [ ] Migration tested on staging with production-volume data
- [ ] Deployment window confirmed (off-peak for HIGH risk)
- [ ] Index files updated after migration: 99-change-log, affected sections

---

## Known Fragile Areas — This Project

| Area | Risk | Reason |
|---|---|---|
| memberships.status column | HIGH | RLS helper functions filter on this — any rename or type change breaks all policies |
| profiles.id = auth.users.id | HIGH | Supabase Auth assumes this 1:1 mapping — deviating from it breaks Auth UI |
| audit_logs (no delete policy) | MEDIUM | If a DELETE policy is accidentally added, audit integrity is compromised |
| tasks.assignee_id (SET NULL) | LOW | Deleting a user silently unassigns their tasks — may surprise users |
