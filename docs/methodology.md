# Methodology

## Database indexing workflow

The recommended workflow is:

```txt
Raw SQL Schema
      ↓
Structured Database Index
      ↓
AI Context Selection
      ↓
Migration Planning
      ↓
Targeted SQL Reading
      ↓
Migration Execution
      ↓
Index Update
```

---

## Index structure

Recommended structure:

```txt
db-index/
├── 00-overview.md
├── 01-tables.md
├── 02-rls-policies.md
├── 03-functions-triggers.md
├── 04-relationships.md
├── 05-migration-risks.md
└── 99-change-log.md
```

---

## Suggested object template

```md
## users

Type: table
Domain: authentication
Purpose: stores authenticated users
Depends on: tenants
Used by: profiles, projects, permissions
RLS impact: HIGH
Migration risk: HIGH
Source file: schema/users.sql
Last updated: 2026-05-08
```

---

## Migration planning workflow

Before any migration:

1. Read overview index
2. Identify affected domain
3. Read related dependency map
4. Inspect RLS impact
5. Inspect triggers/functions impact
6. Classify migration risk
7. Read only required SQL files
8. Create migration plan
9. Execute migration
10. Update index

---

## Migration risk levels

### LOW

Examples:

- additive columns
- new indexes
- optional fields
- isolated tables

### MEDIUM

Examples:

- relationship changes
- constraint changes
- partial policy changes
- trigger updates

### HIGH

Examples:

- tenant isolation changes
- RLS rewrites
- destructive migrations
- column renames
- ownership changes
- auth changes
- large type conversions

---

## Token optimization principles

Do not read:

- full schemas unnecessarily
- historical migrations by default
- unrelated domains
- large SQL dumps without filtering

Prefer:

- indexed summaries
- dependency-driven retrieval
- focused SQL inspection
- incremental context loading

---

## Recommended AI behavior

An AI agent should:

- minimize unnecessary reads
- avoid assumptions
- classify migration risks
- validate RLS implications
- explain dependency impacts
- generate reversible migrations when possible
- update index documentation after changes
