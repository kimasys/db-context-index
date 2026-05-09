# 02 — RLS Policies

**PT-BR:** Índice de políticas de Row Level Security.

---

## Core Principle

RLS (Row Level Security) must be enabled on every table in the `public` schema — no exceptions.

Without RLS, any request using the `anon` key can read all data.

---

## Template — How to document each policy

```md
## policy_name

Table: table_name
Operation: SELECT | INSERT | UPDATE | DELETE | ALL
Role: authenticated | anon | service_role | admin
Using expression: (SQL expression for reads)
With check expression: (SQL expression for writes)
Helper functions: list of helper functions used
Risk: LOW | MEDIUM | HIGH
Notes: edge cases, known issues, related policies
```

---

## Helper Functions

Document reusable authorization functions here before individual policies.

---

### get_user_org_ids()

```sql
create or replace function get_user_org_ids()
returns setof uuid
language sql
security definer
stable
as $$
  select organization_id
  from memberships
  where user_id = auth.uid()
  and status = 'active';
$$;
```

Used by: all org-scoped SELECT policies
Risk: HIGH — changes here affect all org-scoped tables
Notes: security definer runs as the function owner, bypassing RLS on memberships itself

---

### is_org_member(org_id uuid)

```sql
create or replace function is_org_member(org_id uuid)
returns boolean
language sql
security definer
stable
as $$
  select exists (
    select 1 from memberships
    where organization_id = org_id
    and user_id = auth.uid()
    and status = 'active'
  );
$$;
```

Used by: INSERT/UPDATE policies on org-scoped tables
Risk: HIGH

---

### is_org_admin(org_id uuid)

```sql
create or replace function is_org_admin(org_id uuid)
returns boolean
language sql
security definer
stable
as $$
  select exists (
    select 1 from memberships
    where organization_id = org_id
    and user_id = auth.uid()
    and role in ('owner', 'admin')
    and status = 'active'
  );
$$;
```

Used by: DELETE and admin-only UPDATE policies
Risk: HIGH

---

## Policy Index

---

## organizations_select

Table: organizations
Operation: SELECT
Role: authenticated
Using: `is_org_member(id)`
With check: —
Helper functions: is_org_member
Risk: HIGH
Notes: users can only see orgs they belong to

---

## organizations_insert

Table: organizations
Operation: INSERT
Role: authenticated
Using: —
With check: `(created_by = auth.uid())`
Helper functions: —
Risk: MEDIUM
Notes: any authenticated user can create an org; billing limits enforced at app layer

---

## memberships_select

Table: memberships
Operation: SELECT
Role: authenticated
Using: `(user_id = auth.uid() or is_org_admin(organization_id))`
Helper functions: is_org_admin
Risk: HIGH
Notes: members see their own rows; admins see all rows in their orgs

---

## projects_select

Table: projects
Operation: SELECT
Role: authenticated
Using: `(organization_id = any(get_user_org_ids()))`
Helper functions: get_user_org_ids
Risk: MEDIUM

---

## projects_insert

Table: projects
Operation: INSERT
Role: authenticated
With check: `is_org_member(organization_id)`
Helper functions: is_org_member
Risk: MEDIUM

---

## tasks_select

Table: tasks
Operation: SELECT
Role: authenticated
Using: `exists (select 1 from projects p where p.id = project_id and p.organization_id = any(get_user_org_ids()))`
Helper functions: get_user_org_ids
Risk: LOW
Notes: inherits org scope via projects join

---

## audit_logs_select

Table: audit_logs
Operation: SELECT
Role: authenticated
Using: `(organization_id = any(get_user_org_ids()) and is_org_admin(organization_id))`
Helper functions: get_user_org_ids, is_org_admin
Risk: MEDIUM
Notes: only admins can read audit logs — no INSERT/UPDATE/DELETE policies exist (enforced by revoke)

---

## Critical Warnings

- The `service_role` key **bypasses all RLS**. Never expose it client-side.
- Always test policies from the client SDK, not the SQL Editor (SQL Editor runs as `postgres`, bypassing RLS).
- Always index columns referenced in RLS expressions — missing indexes are the top performance killer.
- Never modify helper functions without auditing all policies that depend on them.
- Changes to `memberships` table structure cascade risk to every RLS policy in the system.

---

## Required Indexes for RLS Performance

```sql
create index on memberships (user_id, organization_id, status);
create index on memberships (organization_id, role, status);
create index on projects (organization_id);
create index on tasks (project_id);
create index on audit_logs (organization_id);
```
