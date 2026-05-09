# db-context-index

Context-Oriented Database Navigation for AI Agents.

**PT-BR:** Navegação orientada a contexto de banco de dados para agentes IA.

---

## What this is

AI coding agents have no map of your database. Before answering even a simple pre-migration question, an unguided agent reads every migration file, every type definition, every component that touches the schema — to reconstruct what it needs.

`db-context-index` is a structured information layer that changes how AI agents navigate database context.

Instead of scanning 49 migration files, the agent reads one structured index. It finds what it needs, loads only what matters, and produces a more consistent, better-grounded response.

**PT-BR:** Em vez de escanear 49 arquivos de migration, o agente lê um índice estruturado. Encontra o que precisa, carrega só o que importa, e produz respostas mais consistentes.

---

## Measured results

Tested on a real Supabase production project (49 migrations, 22 tables, full RLS):

| Metric | Without index | With index | Change |
|---|---|---|---|
| Cache read tokens | 235,900 | 94,200 | **▼ 60.1%** |
| Total session cost | $0.2322 | $0.1452 | **▼ 37.5%** |
| Response quality | baseline | more best practices | ↑ |
| API processing time | 1m 10s | 1m 10s | = equal |

→ Full methodology and analysis: [`docs/case-study-claude-code.md`](docs/case-study-claude-code.md)

---

## How it works

```
Without index:
Claude → 49 migrations → types.ts (1,500+ lines) → components → infer patterns → answer
         ════════════════════════════ 235,900 tokens ═══════════════════════════════

With index:
Claude → SCHEMA.md (477 lines) → targeted SQL if needed → answer
         ══════════ 94,200 tokens ═════════════════════
```

The index does not change the prompt. It changes what the agent reads before answering.

---

## Core idea

Do not make the AI read the full database schema by default.

Give it a structured index first:

```txt
db-index/
├── 00-overview.md           → domain map, critical systems
├── 01-tables.md             → table inventory with risk + FK targets
├── 02-rls-policies.md       → all policies and helper functions
├── 03-functions-triggers.md → all functions and triggers
├── 04-relationships.md      → FK graph and cascade rules
├── 05-migration-risks.md    → known fragile areas
└── 99-change-log.md         → migration history with risk classification
```

The index is the navigation layer. The agent reads it, identifies what matters, and opens only the SQL files it actually needs.

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
│   ├── roadmap.md
│   └── case-study-claude-code.md   ← real benchmark, real project
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

Install the skill at:

```txt
skills/db-context-indexer/SKILL.md
```

Copy it to `.claude/commands/db-context-indexer.md` in your project and invoke with `/db-context-indexer` before any database work.

Use it when working with: PostgreSQL schemas, Supabase migrations, RLS policies, triggers, functions, views, or schema refactoring.

---

## Why "context engineering" and not "prompt engineering"

A prompt tells the agent *what to do*.

A context index changes *what the agent reads before doing it*.

The two tests in this project had nearly identical direct input tokens (352 vs 349) — the same prompt, the same system. What changed was the reading behavior: which files Claude opened, in what order, and how much it loaded before it could answer with confidence.

This is closer to **context architecture** than to prompt engineering.

The structured index does three things a prompt cannot:
1. **Eliminates search** — patterns are documented, not inferred from 49 files
2. **Externalizes institutional knowledge** — RLS patterns, risk areas, and dependencies become first-class documentation
3. **Creates behavioral consistency** — every session follows the same workflow regardless of which files happen to be in the agent's cache

---

## Status

v1 — documentation-first, validated with a real production project.

Benchmark published. Methodology documented. Ready to use.

---

## License

MIT License.
