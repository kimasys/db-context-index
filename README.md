# db-context-index

Structured database context indexing for AI-assisted SQL migrations.

**PT-BR:** Indexação estruturada de contexto de banco de dados para migrations SQL assistidas por IA.

---

## What this project solves

AI coding agents are powerful, but database work is context-sensitive. When a PostgreSQL or Supabase schema grows, agents often need to read large SQL files before making even a small migration. That increases token usage, slows down the workflow, and creates a real risk of missing the specific relationship, RLS policy, trigger, or function that matters.

`db-context-index` proposes a practical documentation and skill pattern for giving AI agents a compact, navigable database map before they touch raw SQL.

**PT-BR:** Este projeto cria uma camada de índice entre a IA e o banco real, permitindo que o agente leia primeiro um mapa compacto e só depois abra os arquivos SQL realmente necessários.

---

## Core idea

Do not make the AI read the full database schema by default.

Make the AI read a structured database index first:

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

The index acts as a navigation layer. It tells the AI where the relevant database context is, what objects are connected, and what risks must be checked before writing a migration.

---

## Main goals

- Reduce token usage when working with large schemas.
- Improve speed when AI agents inspect database context.
- Reduce migration errors caused by missing dependencies.
- Make RLS and multi-tenant rules easier to reason about.
- Keep database knowledge reusable across AI sessions.
- Give humans and agents the same structured map of the database.

---

## Current focus

This first version focuses on:

- PostgreSQL
- Supabase
- SQL migrations
- RLS policies
- functions and triggers
- schema dependency mapping
- Claude Code Skills

---

## Repository structure

```txt
.
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── docs/
│   ├── concept.md
│   ├── methodology.md
│   ├── token-efficiency.md
│   └── roadmap.md
├── examples/
│   └── supabase/
│       └── db-index/
│           ├── 00-overview.md
│           ├── 01-tables.md
│           ├── 02-rls-policies.md
│           ├── 03-functions-triggers.md
│           ├── 04-relationships.md
│           ├── 05-migration-risks.md
│           └── 99-change-log.md
└── skills/
    └── db-context-indexer/
        ├── SKILL.md
        └── references/
            ├── migration-safety-checklist.md
            ├── postgres-best-practices.md
            └── supabase-rls-checklist.md
```

---

## Claude Code Skill

The first skill lives at:

```txt
skills/db-context-indexer/SKILL.md
```

Use it when working with PostgreSQL, Supabase, SQL migrations, RLS policies, triggers, functions, views, or schema refactoring.

---

## Status

Experimental methodology based on a real-world AI-assisted development pain point.

The first public version is intentionally documentation-first. Automation scripts may be added later.

---

## License

MIT License.
