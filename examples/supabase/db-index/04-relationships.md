# 04 — Relationships

**PT-BR:** Mapa de relacionamentos e dependências entre tabelas.

---

## Purpose

This file maps foreign key relationships, cascade rules, and logical dependency chains.

It answers: **"If I change X, what else breaks?"**

---

## Dependency Graph

```
auth.users (Supabase managed)
  └── profiles (1:1, CASCADE DELETE)
  └── memberships (1:N, RESTRICT)
  └── audit_logs (1:N, SET NULL on delete)

organizations
  └── memberships (1:N, CASCADE DELETE)
  └── projects (1:N, RESTRICT)
  └── audit_logs (1:N, SET NULL on delete)

memberships
  └── (no dependents — terminal node for org membership resolution)

projects
  └── tasks (1:N, CASCADE DELETE)
  └── audit_logs (1:N, SET NULL on delete)

tasks
  └── (no dependents currently)

audit_logs
  └── (terminal node — nothing depends on it)
```

---

## Foreign Key Inventory

| FK | From | To | On Delete | On Update |
|---|---|---|---|---|
| profiles.id | profiles | auth.users | CASCADE | CASCADE |
| memberships.user_id | memberships | users | RESTRICT | CASCADE |
| memberships.organization_id | memberships | organizations | CASCADE | CASCADE |
| projects.organization_id | projects | organizations | RESTRICT | CASCADE |
| projects.created_by | projects | users | SET NULL | CASCADE |
| tasks.project_id | tasks | projects | CASCADE | CASCADE |
| tasks.assignee_id | tasks | users | SET NULL | CASCADE |
| audit_logs.user_id | audit_logs | users | SET NULL | CASCADE |
| audit_logs.organization_id | audit_logs | organizations | SET NULL | CASCADE |

---

## Cascade Risk Analysis

### CASCADE DELETE chains (highest risk)

```
organizations → memberships → (all org membership context lost)
projects → tasks → (all task data lost when project deleted)
auth.users → profiles → (profile deleted when user deleted)
```

**Action required before deletion:** confirm application-level soft-delete or archival is in place before relying on CASCADE DELETE in production.

---

### RESTRICT chains (intentional blockers)

```
users cannot be deleted if they have active memberships
organizations cannot be deleted if they have active projects
```

These are intentional guards. If you need to delete a user or org, you must clean up memberships and projects first.

---

## Logical Dependency Map (beyond foreign keys)

Some dependencies are enforced by RLS policy logic rather than FK constraints:

| Dependency | Type | Risk if broken |
|---|---|---|
| RLS policies → memberships.status | Logical | HIGH — stale status bypasses isolation |
| RLS helper functions → memberships schema | Logical | HIGH — schema change breaks all policies |
| handle_new_user → profiles schema | Logical | HIGH — auth trigger fails silently |
| log_mutation → audit_logs schema | Logical | MEDIUM — audit trail breaks |

---

## Domain Isolation Map

```
┌─────────────────────────────────────────────────┐
│ AUTHENTICATION DOMAIN                           │
│  auth.users → profiles                          │
├─────────────────────────────────────────────────┤
│ ORGANIZATION DOMAIN                             │
│  organizations → memberships                    │
│  (memberships bridges auth ↔ org domains)       │
├─────────────────────────────────────────────────┤
│ OPERATIONS DOMAIN                               │
│  projects → tasks                               │
│  (all scoped via organization_id)               │
├─────────────────────────────────────────────────┤
│ SYSTEM DOMAIN                                   │
│  audit_logs (receives events from all domains)  │
└─────────────────────────────────────────────────┘
```

---

## Migration Checklist — Relationship Changes

Before modifying any relationship:

- [ ] Identify all downstream FK dependents
- [ ] Identify all RLS policies that reference this table
- [ ] Identify all trigger functions that touch this table
- [ ] Check cascade behavior — is CASCADE DELETE intentional?
- [ ] Check RESTRICT constraints — will deletion be blocked?
- [ ] Verify that no logical dependencies (RLS helpers, triggers) break
- [ ] Plan rollback: can this migration be reversed?
- [ ] Run on a staging database with production-equivalent data volume first
