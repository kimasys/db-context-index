# 03 — Functions and Triggers

**PT-BR:** Índice de funções e triggers do banco de dados.

---

## Template — Function

```md
## function_name()

Type: function | trigger function | security definer function
Domain: authentication | organization | operations | system
Purpose: what this function does
Inputs: parameter list
Returns: return type
Called by: list of triggers, policies, or application code
Depends on: tables, other functions
Risk: LOW | MEDIUM | HIGH
Notes: edge cases, security implications
Source file: path/to/file.sql
Last updated: YYYY-MM-DD
```

---

## Template — Trigger

```md
## trigger_name

Table: table_name
Event: BEFORE | AFTER — INSERT | UPDATE | DELETE | TRUNCATE
For each: ROW | STATEMENT
Function: trigger_function_name()
Purpose: what this trigger does
Risk: LOW | MEDIUM | HIGH
Notes: order dependencies, known conflicts
```

---

## Functions

---

### handle_new_user()

Type: trigger function
Domain: authentication
Purpose: creates a profile row when a new user registers via Supabase Auth
Inputs: —
Returns: trigger
Called by: on_auth_user_created (trigger on auth.users)
Depends on: profiles table
Risk: HIGH
Notes: runs on auth.users which is Supabase-managed — changes here require careful coordination with Supabase Auth behavior
Source file: schema/auth/handle_new_user.sql
Last updated: 2026-05-08

```sql
create or replace function handle_new_user()
returns trigger
language plpgsql
security definer
set search_path = public
as $$
begin
  insert into public.profiles (id, email, created_at)
  values (new.id, new.email, now());
  return new;
end;
$$;
```

---

### handle_updated_at()

Type: trigger function
Domain: system
Purpose: automatically sets updated_at to now() on any row update
Inputs: —
Returns: trigger
Called by: multiple triggers across tables (see trigger index below)
Depends on: —
Risk: LOW
Notes: generic utility — safe to apply to new tables
Source file: schema/system/handle_updated_at.sql
Last updated: 2026-05-08

```sql
create or replace function handle_updated_at()
returns trigger
language plpgsql
as $$
begin
  new.updated_at = now();
  return new;
end;
$$;
```

---

### create_default_org_membership()

Type: function
Domain: organization
Purpose: inserts an owner membership when a new organization is created
Inputs: org_id uuid, user_id uuid
Returns: void
Called by: application layer (post-insert hook)
Depends on: memberships table
Risk: HIGH
Notes: if this fails, user creates an org they cannot access; wrap in transaction at app layer
Source file: schema/org/create_default_org_membership.sql
Last updated: 2026-05-08

---

### log_mutation()

Type: trigger function
Domain: system
Purpose: appends a record to audit_logs on any INSERT/UPDATE/DELETE
Inputs: —
Returns: trigger
Called by: audit triggers on users, memberships, projects
Depends on: audit_logs table
Risk: MEDIUM
Notes: high-volume trigger — monitor write performance on large tables
Source file: schema/system/log_mutation.sql
Last updated: 2026-05-08

---

## Triggers

---

### on_auth_user_created

Table: auth.users (Supabase managed)
Event: AFTER INSERT
For each: ROW
Function: handle_new_user()
Purpose: sync new Supabase Auth user to public.profiles
Risk: HIGH
Notes: cannot be modified without Supabase dashboard access; auth.users schema is not directly editable

---

### set_profiles_updated_at

Table: profiles
Event: BEFORE UPDATE
For each: ROW
Function: handle_updated_at()
Purpose: maintain updated_at timestamp
Risk: LOW

---

### set_projects_updated_at

Table: projects
Event: BEFORE UPDATE
For each: ROW
Function: handle_updated_at()
Purpose: maintain updated_at timestamp
Risk: LOW

---

### audit_memberships

Table: memberships
Event: AFTER INSERT OR UPDATE OR DELETE
For each: ROW
Function: log_mutation()
Purpose: write all membership changes to audit_logs
Risk: MEDIUM
Notes: membership changes are security-critical events; always audit them

---

## Critical Notes

- Functions with `security definer` run as the function owner (usually `postgres`) — they bypass RLS. Audit them carefully.
- Always set `search_path = public` in security definer functions to prevent search_path hijacking.
- Triggers on `auth.users` are Supabase-managed and may be affected by Supabase Auth version updates.
- Never drop a trigger function without first dropping the trigger that calls it — PostgreSQL will block the drop if dependencies exist.
