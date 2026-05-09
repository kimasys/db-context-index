# CLAUDE.md — db-context-index

## Project Summary

`db-context-index` is a structured methodology and skill pattern for giving AI agents a compact, navigable database map before they touch raw SQL.

**Problem:** AI agents working on large PostgreSQL/Supabase schemas waste tokens reading entire schemas, slow down migrations, and risk missing critical dependencies (RLS policies, triggers, functions, relationships).

**Solution:** A 7-file indexed database context layer that the agent reads first, then loads only the SQL files it actually needs.

---

## Repository Structure

```
db-context-index/
├── CLAUDE.md                          ← you are here
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── docs/
│   ├── concept.md                     ← core philosophy
│   ├── methodology.md                 ← how the index is built
│   ├── token-efficiency.md
│   └── roadmap.md
├── examples/
│   └── supabase/
│       └── db-index/                  ← reference index files
│           ├── 00-overview.md
│           ├── 01-tables.md
│           ├── 02-rls-policies.md
│           ├── 03-functions-triggers.md
│           ├── 04-relationships.md
│           ├── 05-migration-risks.md
│           └── 99-change-log.md
├── skills/
│   └── db-context-indexer/
│       ├── SKILL.md                   ← Claude Code skill definition
│       └── references/
│           ├── migration-safety-checklist.md
│           ├── postgres-best-practices.md
│           └── supabase-rls-checklist.md
└── vault/                             ← project work log and decision history
```

---

## Core Principle

**Never read the full schema by default.**

Always inspect the database index first. Only open raw SQL files when the indexed context indicates deeper inspection is required.

---

## Standard Index Structure (7 files)

Every project using this methodology creates a `db-index/` folder with:

| File | Purpose |
|---|---|
| `00-overview.md` | High-level domain map, critical systems, key dependencies |
| `01-tables.md` | Table list, row counts, ownership, key columns |
| `02-rls-policies.md` | All RLS policies, affected roles, helper functions |
| `03-functions-triggers.md` | All functions and triggers with descriptions |
| `04-relationships.md` | Foreign keys, cascades, dependency chains |
| `05-migration-risks.md` | Known risks, fragile areas, required checks |
| `99-change-log.md` | Migration history with risk classification |

---

## Workflow Before Any Migration

1. Read `00-overview.md` — identify the affected domain
2. Read the relevant section files (tables, RLS, functions)
3. Inspect dependency and RLS impact
4. Classify migration risk (LOW / MEDIUM / HIGH)
5. Open only the required SQL files
6. Create and execute the migration plan
7. Update index files after the change

---

## Migration Risk Classification

| Level | Triggers |
|---|---|
| LOW | Additive changes, isolated indexes, optional fields |
| MEDIUM | Relationship changes, trigger modifications, constraint updates |
| HIGH | RLS rewrites, tenant isolation changes, destructive migrations, auth-related changes, type conversions on production data |

---

## Technology Stack

- **Database:** PostgreSQL, Supabase
- **Focus areas:** RLS policies, functions, triggers, views, SQL migrations
- **AI integration:** Claude Code Skills
- **License:** MIT

---

## Claude Code Skill

The skill is defined at `skills/db-context-indexer/SKILL.md`.

Trigger it when working with: PostgreSQL schemas, Supabase migrations, RLS policies, triggers, functions, views, or schema refactoring.

---

## Output Format for Migration Work

Always structure responses as:

```
## Context Used
## Objects Affected
## Dependency Impact
## RLS Impact
## Migration Risk
## Migration Plan
## SQL Changes
## Validation Checklist
## Required Index Updates
```

---

## Vault

Project decisions, session logs, and incremental documentation are tracked in `vault/`.

See `vault/README.md` for the vault structure and usage guide.

---

## Status

Experimental. Documentation-first. Automation scripts may be added in future iterations.
