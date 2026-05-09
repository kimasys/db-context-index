# PostgreSQL Best Practices for AI-Assisted Development

**PT-BR:** Melhores práticas PostgreSQL para desenvolvimento assistido por IA.

---

## Schema Design

### Naming conventions

- Use `snake_case` for all identifiers (tables, columns, functions)
- Use descriptive plural names for tables: `users`, `projects`, `memberships`
- Prefix trigger functions with the action: `handle_new_user`, `set_updated_at`
- Prefix indexes with `idx_`: `idx_users_email`, `idx_tasks_project_id`
- Prefix RLS policies with table + operation: `users_select`, `projects_insert`

---

### Column standards

Always include on every table:

```sql
id          uuid primary key default gen_random_uuid()
created_at  timestamptz not null default now()
updated_at  timestamptz not null default now()
```

For Supabase tables exposed to tenants:

```sql
organization_id  uuid not null references organizations(id)
created_by       uuid references auth.users(id)
```

---

### Data types

| Use case | Recommended type |
|---|---|
| Primary keys | `uuid` (not serial) |
| Timestamps | `timestamptz` (always timezone-aware) |
| Money/currency | `numeric(19,4)` (never float) |
| Short text | `text` (PostgreSQL text is efficient — avoid varchar) |
| Status/enum | `text` with CHECK constraint or PostgreSQL `enum` |
| JSON data | `jsonb` (not json — jsonb is indexed and faster) |
| Flags | `boolean not null default false` |

---

## Indexing Strategy

### Required indexes (always create)

```sql
-- Every FK column should be indexed
create index on tasks (project_id);
create index on memberships (user_id);
create index on memberships (organization_id);

-- Columns referenced in RLS expressions
create index on memberships (user_id, organization_id, status);

-- Columns used in ORDER BY or WHERE clauses
create index on projects (organization_id, created_at desc);
```

### Partial indexes (for filtered queries)

```sql
-- Only index active memberships (most queries filter on status='active')
create index on memberships (user_id, organization_id)
where status = 'active';
```

### When NOT to index

- Columns with very low cardinality (e.g., boolean flags on small tables)
- Columns never used in WHERE, JOIN, or ORDER BY
- Tables with <1000 rows (sequential scan is faster)

---

## Query Performance

### EXPLAIN ANALYZE

Always run before deploying queries on large tables:

```sql
explain (analyze, buffers, format text)
select * from tasks
where project_id = $1
order by created_at desc
limit 20;
```

Look for: `Seq Scan` on large tables (missing index), high `actual rows` vs `estimated rows` (stale statistics), high `Buffers: shared hit` (memory pressure).

### Run ANALYZE after large data changes

```sql
analyze tasks;
-- or for full database
analyze;
```

PostgreSQL autovacuum handles this, but after bulk inserts or migrations it is worth running manually.

---

## Functions

### Security definer pattern

```sql
create or replace function my_function()
returns void
language plpgsql
security definer
set search_path = public  -- always set search_path in security definer functions
as $$
begin
  -- function body
end;
$$;
```

**Always set `search_path = public`** in security definer functions. Without it, a malicious user could create a function in another schema with the same name as a system function and hijack execution.

---

### Idempotent function creation

Always use `create or replace`:

```sql
create or replace function my_function() ...
```

This allows re-running migration files without error if the function already exists.

---

## Migrations

### Reversible-first design

For every migration, write the rollback SQL before applying it:

```sql
-- migration: add column
alter table tasks add column priority text;

-- rollback:
alter table tasks drop column priority;
```

### Version tracking

Use a migrations table to track applied migrations:

```sql
create table if not exists _migrations (
  id serial primary key,
  name text not null unique,
  applied_at timestamptz not null default now()
);
```

Or use Supabase's built-in migration system via the CLI.

### Never modify applied migrations

Once a migration is applied to production, it is immutable. Create a new migration to fix it — never edit the original.

---

## Supabase-Specific Patterns

### Auth integration

```sql
-- Always reference auth.users, not a custom users table
auth.uid()         -- current user's UUID
auth.jwt()         -- current user's full JWT claims
auth.role()        -- current user's role (authenticated, anon, service_role)
```

### Custom JWT claims for tenant context

Set custom claims via Auth hooks:

```sql
-- In a Supabase Auth hook function:
select jsonb_build_object(
  'org_id', (select organization_id from memberships where user_id = uid limit 1)
) into claims;
```

Access in RLS:

```sql
(auth.jwt() ->> 'org_id')::uuid = organization_id
```

This avoids a subquery to `memberships` on every row evaluation — critical for performance.

---

## Connection Management

- Use connection pooling (PgBouncer in Supabase) for serverless/edge functions
- Avoid long-lived transactions — they hold locks and bloat WAL
- Use `statement_timeout` in application-layer queries to prevent runaway queries:

```sql
set statement_timeout = '30s';
```

---

## Documentation Standards for AI Agents

Every table should have a PostgreSQL COMMENT:

```sql
comment on table tasks is 'Work items within a project. Scoped to an organization via the projects relationship.';
comment on column tasks.priority is 'LOW, NORMAL, HIGH, or URGENT. Defaults to NORMAL.';
```

These comments are visible to AI agents via `information_schema` and `pg_description` — they reduce the need for external documentation.

```sql
-- AI agents can query this to understand the schema
select
  t.table_name,
  c.column_name,
  pgd.description
from information_schema.tables t
join information_schema.columns c on c.table_name = t.table_name
left join pg_description pgd on pgd.objoid = (t.table_schema || '.' || t.table_name)::regclass
  and pgd.objsubid = c.ordinal_position
where t.table_schema = 'public'
order by t.table_name, c.ordinal_position;
```
