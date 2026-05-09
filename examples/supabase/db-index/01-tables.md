# 01 — Tables

**PT-BR:** Índice de tabelas do banco de dados.

---

## Template — How to document each table

```md
## table_name

Type: table | view | materialized_view
Domain: authentication | organization | operations | analytics | system
Purpose: one-line description of what this table stores
Owner: service or team responsible
Row estimate: approximate row count
Depends on: list of tables this table depends on (foreign keys pointing outward)
Used by: list of tables, functions, or services that reference this table
RLS enabled: yes | no
RLS impact: LOW | MEDIUM | HIGH
Migration risk: LOW | MEDIUM | HIGH
Source file: path/to/schema/file.sql
Last updated: YYYY-MM-DD
```

---

## Example Index — Supabase SaaS Platform

---

## users

Type: table
Domain: authentication
Purpose: stores authenticated user accounts, mirrors auth.users
Owner: auth service
Row estimate: 10k–100k
Depends on: auth.users (Supabase managed)
Used by: profiles, memberships, audit_logs
RLS enabled: yes
RLS impact: HIGH
Migration risk: HIGH
Source file: schema/auth/users.sql
Last updated: 2026-05-08

---

## profiles

Type: table
Domain: authentication
Purpose: extended user data — display name, avatar, preferences
Owner: user service
Row estimate: 10k–100k
Depends on: users
Used by: memberships, projects, comments
RLS enabled: yes
RLS impact: MEDIUM
Migration risk: MEDIUM
Source file: schema/auth/profiles.sql
Last updated: 2026-05-08

---

## organizations

Type: table
Domain: organization
Purpose: tenant root entity — every isolated workspace belongs to an org
Owner: org service
Row estimate: 1k–10k
Depends on: —
Used by: memberships, projects, billing, invitations
RLS enabled: yes
RLS impact: HIGH
Migration risk: HIGH
Source file: schema/org/organizations.sql
Last updated: 2026-05-08

---

## memberships

Type: table
Domain: organization
Purpose: maps users to organizations with a role
Owner: org service
Row estimate: 10k–100k
Depends on: users, organizations
Used by: all org-scoped resources (RLS policies)
RLS enabled: yes
RLS impact: HIGH
Migration risk: HIGH
Source file: schema/org/memberships.sql
Last updated: 2026-05-08

---

## projects

Type: table
Domain: operations
Purpose: main work unit scoped to an organization
Owner: project service
Row estimate: 5k–50k
Depends on: organizations, users (created_by)
Used by: tasks, comments, attachments
RLS enabled: yes
RLS impact: MEDIUM
Migration risk: MEDIUM
Source file: schema/ops/projects.sql
Last updated: 2026-05-08

---

## tasks

Type: table
Domain: operations
Purpose: items within a project, assignable to users
Owner: project service
Row estimate: 50k–500k
Depends on: projects, users (assignee)
Used by: comments, attachments, audit_logs
RLS enabled: yes
RLS impact: LOW
Migration risk: LOW
Source file: schema/ops/tasks.sql
Last updated: 2026-05-08

---

## audit_logs

Type: table
Domain: system
Purpose: append-only record of mutations across the platform
Owner: system
Row estimate: 1M+
Depends on: users, organizations
Used by: — (terminal table, nothing depends on it)
RLS enabled: yes (read-only for admins)
RLS impact: MEDIUM
Migration risk: HIGH (volume + immutability requirements)
Source file: schema/system/audit_logs.sql
Last updated: 2026-05-08

---

## Key Notes

- All tables exposed to Supabase's `public` schema must have RLS enabled.
- Any table without RLS is a data exposure risk for anon or authenticated roles.
- `memberships` is the most critical dependency — most RLS policies check it to resolve org membership.
- `audit_logs` must never be updated or deleted — policies must enforce insert-only.
