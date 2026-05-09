# Project History — db-context-index

**PT-BR:** Histórico completo do projeto, desde a ideia inicial até o estado atual.

---

## Origin — The Real Pain Point

The project started from a real frustration encountered during AI-assisted development with PostgreSQL and Supabase.

The problem was simple but costly: every time an AI coding agent was asked to help with a database migration — even a small one — it would start by reading large SQL files, schema dumps, and migration history. For a growing schema with 30–50 tables, full RLS policies, functions, and triggers, this pre-work consumed 30,000–80,000 tokens before the agent wrote a single line of SQL.

The consequences were real:
- Slower sessions (more tokens = more latency)
- Higher cost per session
- Higher risk of errors — agents missing a dependency, a policy, or a trigger they didn't fully load
- Inconsistent behavior between sessions — the agent would "forget" context and re-derive it differently each time

---

## The Insight

The core insight came from a question: *what would a senior database engineer do before writing a migration?*

They would not read every SQL file. They would consult a map — a mental model of the schema. They would ask: which domain is affected, what are the dependencies, what are the RLS implications, what is fragile?

The answer was to create that map explicitly, as a structured set of files, so the AI agent could do the same thing a senior engineer does: consult the map first, load only the relevant detail second.

**The db-index pattern was born: 7 files that act as a navigable database map.**

---

## The Methodology

```
Raw SQL Schema
      ↓
Structured Database Index (7 files)
      ↓
AI Context Selection (read only what matters)
      ↓
Migration Planning
      ↓
Targeted SQL Reading
      ↓
Migration Execution
      ↓
Index Update (keep the map current)
```

The 7-file structure covers the full surface area of a PostgreSQL/Supabase schema without redundancy:

| File | What it maps |
|---|---|
| `00-overview.md` | Domain map, critical systems, key dependencies |
| `01-tables.md` | Table inventory with ownership, risk, and dependencies |
| `02-rls-policies.md` | All RLS policies and helper functions |
| `03-functions-triggers.md` | All functions and triggers |
| `04-relationships.md` | FK graph, cascade rules, logical dependencies |
| `05-migration-risks.md` | Known fragile areas and risk patterns |
| `99-change-log.md` | Migration history with risk classification |

---

## Technology Choices

**PostgreSQL** — the most widely used open-source database, with a rich feature set for RLS, functions, triggers, and schema management.

**Supabase** — the most common PostgreSQL-as-a-service platform for developers building SaaS products. Its RLS + Auth model adds complexity that benefits greatly from structured indexing.

**Claude Code Skills** — the skill format allows the methodology to be packaged as a reusable, triggerable behavior. An agent with the skill installed follows the index-first pattern automatically, without the developer needing to prompt it every time.

**MIT License** — chosen to maximize adoption and contribution. The methodology is a pattern, not a product. Open is the right model.

---

## Broader Vision — Skills Architecture

`db-context-index` is designed as the first module of a larger architecture.

The long-term vision is a suite of Claude Code skills — one per domain of a large digital platform project — that together give an AI agent a complete, structured, token-efficient view of an entire codebase.

Planned domains:

| Domain | Skill | Purpose |
|---|---|---|
| Database | `db-context-indexer` | Schema, RLS, migrations |
| Backend | `api-context-indexer` | Endpoints, services, business logic |
| Authentication | `auth-context-indexer` | Auth flows, permissions, session logic |
| Frontend | `ui-context-indexer` | Components, state, routing |
| Infrastructure | `infra-context-indexer` | Cloud resources, CI/CD, env config |
| Testing | `test-context-indexer` | Test coverage, risk areas, regression map |

Each skill follows the same architecture:
- Structured index files for the domain
- Token-efficient context loading rules
- Risk classification for changes
- AI agent workflow guidance

The aggregate result: an AI agent that navigates a full platform project the way a senior engineer would — domain by domain, depth on demand.

---

## Personal Context — Why This Matters

This project is also part of a personal transition.

The author — a non-developer building production software with AI assistance — hit these database friction points repeatedly across real projects. The methodology was developed not as theory, but as a solution to actual blockers in day-to-day AI-assisted work.

Publishing this as an open-source project serves two purposes:

1. **Community value** — other developers hitting the same pain point can use and improve the pattern.
2. **Professional signal** — a public proof of technical thinking, AI-assisted engineering methodology, and practical problem-solving. Part of building a foundation for a transition toward developer work.

---

## Timeline

| Date | Milestone |
|---|---|
| Pre-project | Pain point identified during real AI-assisted database work |
| 2026-05-08 | Repository created on GitHub (kimasys/db-context-index) |
| 2026-05-08 | CLAUDE.md and vault structure created |
| 2026-05-08 | All 7 db-index example files created |
| 2026-05-08 | All docs files completed |
| 2026-05-08 | All 3 skill reference files created |
| 2026-05-08 | Project history compiled |
| — | v1 complete — ready for public visibility |

---

## Status

v1 is documentation-first and complete. The methodology works today without any automation. A developer or AI agent can follow it immediately.

Next: validate the skill on a real migration use case before any external communication.
