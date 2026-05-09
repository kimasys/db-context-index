# Supabase RLS Checklist

**PT-BR:** Checklist de Row Level Security para Supabase com foco em multi-tenant.

---

## Foundational Rules

These are non-negotiable for any Supabase production deployment:

- [ ] RLS is enabled on every table in the `public` schema
- [ ] No table is accessible to `anon` role unless explicitly intended
- [ ] Service role key is never exposed in client-side code
- [ ] All RLS policies are tested from the client SDK, not the SQL Editor

```sql
-- Verify RLS status on all public tables
select
  schemaname,
  tablename,
  rowsecurity
from pg_tables
where schemaname = 'public'
order by tablename;
-- rowsecurity must be 't' for all rows
```

---

## Policy Architecture

### Minimum policy set per table

| Operation | Required? | Notes |
|---|---|---|
| SELECT | Yes | Without this, authenticated users see no rows |
| INSERT | Yes (if writable) | Without this, authenticated users cannot insert |
| UPDATE | Yes (if mutable) | Without this, authenticated users cannot update |
| DELETE | Only if needed | Prefer soft deletes — avoid exposing DELETE |

**A table with RLS enabled but no SELECT policy returns zero rows to all users.** This is a common silent failure.

---

### Policy types

```sql
-- PERMISSIVE (default): policies are OR'd together
-- Any permissive policy that allows access grants access
create policy "users_own_rows" on tasks
  for select to authenticated
  using (created_by = auth.uid());

-- RESTRICTIVE: policies are AND'd with permissive policies
-- The row must pass both permissive AND all restrictive policies
create policy "org_scope" on tasks
  as restrictive
  for all to authenticated
  using (organization_id = any(get_user_org_ids()));
```

Use `RESTRICTIVE` for tenant isolation — it cannot be bypassed by other permissive policies.

---

## Multi-Tenant Isolation Pattern

### Option A — tenant_id column (recommended for most projects)

```sql
-- Every tenant-scoped table has organization_id
alter table projects add column organization_id uuid not null
  references organizations(id);

-- RLS policy enforces isolation
create policy "org_isolation" on projects
  as restrictive
  for all to authenticated
  using (organization_id = any(get_user_org_ids()));
```

### Option B — JWT claim (recommended for performance at scale)

Set `org_id` as a custom claim in the JWT:

```sql
-- Read from JWT instead of querying memberships on every row
create policy "org_isolation" on projects
  for all to authenticated
  using (organization_id = (auth.jwt() ->> 'org_id')::uuid);
```

**Performance difference:** Option A runs a subquery against `memberships` for every row evaluated. Option B reads from the in-memory JWT. For tables with >100k rows, Option B is significantly faster.

---

## Policy Testing Protocol

### Test matrix (run for every policy change)

| Scenario | Expected result |
|---|---|
| anon user reads table | 0 rows OR error |
| authenticated user reads own org's rows | correct rows only |
| authenticated user reads another org's rows | 0 rows |
| authenticated user inserts into own org | success |
| authenticated user inserts into another org | error |
| admin user reads all org rows | correct rows |
| service_role reads all rows | all rows (bypasses RLS — expected) |

### How to test from psql or Supabase SQL Editor

```sql
-- Simulate an authenticated user with a specific JWT
begin;
  set local role authenticated;
  set local request.jwt.claims to '{"sub": "user-uuid-here", "role": "authenticated"}';
  
  -- Your test query here
  select * from projects;
  
rollback; -- always rollback test sessions
```

### How to test from client SDK (preferred)

```javascript
const { data, error } = await supabase
  .from('projects')
  .select('*')
// Must use a regular user JWT, not service_role
```

---

## Performance Checklist

Every column referenced in a RLS `using` or `with check` expression must be indexed:

```sql
-- Required indexes for the example policies
create index on memberships (user_id, organization_id, status);
create index on projects (organization_id);
create index on tasks (project_id);
create index on audit_logs (organization_id);
```

Without these indexes, RLS evaluation performs a sequential scan of `memberships` for every row returned by any query — catastrophic at scale.

---

## Common RLS Mistakes

### 1. Missing SELECT policy on a new table

Symptom: queries return empty results even though rows exist.
Fix: add a SELECT policy.

```sql
-- Symptom
select * from new_table; -- returns 0 rows
-- Fix
create policy "new_table_select" on new_table
  for select to authenticated
  using (organization_id = any(get_user_org_ids()));
```

---

### 2. Testing in SQL Editor (bypasses RLS)

Symptom: policies appear to work in testing but fail in the application.
Fix: always test from the client SDK or with explicit role switching.

---

### 3. Missing index on RLS column

Symptom: query latency spikes after enabling RLS.
Fix: add the missing index.

```sql
explain analyze select * from tasks where true;
-- If you see "Seq Scan on memberships" inside the plan, you're missing an index
```

---

### 4. service_role key exposed on the client

Symptom: all RLS bypassed in production — all users can see all data.
Fix: rotate the service_role key immediately, audit application code, move service_role calls to server-side only.

---

### 5. Policy grants more than intended (PERMISSIVE overlap)

Multiple permissive policies are OR'd. A row is returned if ANY permissive policy allows it.

```sql
-- Dangerous: this policy allows any authenticated user to read everything
create policy "debug_all" on projects
  for select to authenticated
  using (true); -- always true = always allowed
```

Always audit all policies on a table — a forgotten debug policy can be a data leak.

---

## Supabase Auth Role Reference

| Role | Description | RLS behavior |
|---|---|---|
| `anon` | Unauthenticated requests | Policies apply, `auth.uid()` returns null |
| `authenticated` | Signed-in users | Policies apply, `auth.uid()` returns user UUID |
| `service_role` | Server-side admin | **Bypasses all RLS** |
| `postgres` | Database superuser | **Bypasses all RLS** |

The SQL Editor runs as `postgres` by default — this is why testing there is unreliable.
