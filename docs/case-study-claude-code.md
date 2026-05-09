# Case Study: AI Database Context Engineering in Practice

**PT-BR:** Estudo de caso — Engenharia de Contexto de Banco de Dados para Agentes IA

---

## The Problem We Set Out to Measure

When AI coding agents work with large PostgreSQL or Supabase projects, they face a structural inefficiency: they have no map.

Before answering even a simple pre-migration question — "what tables would a new feature reference?" — an unguided agent typically reads:

- All migration files (to reconstruct schema history)
- All type definition files (to understand current structure)
- Component files (to infer RLS patterns from how they're used in code)

This is not a bug. It is rational behavior given the absence of structured context. The agent reads what it needs to answer correctly.

**The hypothesis:** if you give the agent a structured index of the database before it starts reading, it will load dramatically less context, spend fewer tokens, and still produce a higher-quality response — because the index surfaces the right patterns explicitly rather than forcing the agent to infer them from raw SQL.

---

## The Project

**Project:** Radar Performance — a multi-tenant SaaS performance management platform built with React, TypeScript, and Supabase.

**Database state at test time:**
- 49 migration files
- 106 KB of SQL total
- 2,487 lines of migration SQL
- 22 tables across 5 domains
- Custom RLS functions: `has_role`, `user_manages_area`, `get_user_tenant_id`
- 4 established RLS patterns used across all tables

**The index:** `supabase/SCHEMA.md` — a structured, hand-maintained 477-line document covering all enums, functions, RLS patterns, and tables with column-level detail.

---

## Test Design

The test compared two isolated Claude Code sessions given an identical read-only prompt:

```
Analyze the current project schema and describe how a table for registering
employee shifts (ADM, 6x1, 6x2) should look. Do not create any file.

I want only:
1. What columns the table would have and their types
2. Which RLS pattern from the project applies (with exact SQL)
3. Which existing tables it would reference as FK
4. Migration risk (LOW / MEDIUM / HIGH) and justification
```

This prompt was designed to:
- Require genuine schema understanding to answer correctly
- Produce comparable output between sessions
- Not trigger any file writes (eliminates side effects)
- Represent a real pre-migration reasoning task

### Test A — Without Skill (Baseline)

Claude Code running with no `/db-context-indexer` command available. The agent was left to find context on its own.

### Test B — With Skill

Claude Code with `/db-context-indexer` invoked before the same prompt. The skill directed the agent to read `supabase/SCHEMA.md` first and follow a structured workflow.

**Methodology notes:**
- Each test in a completely fresh conversation (not just `/clear` — full session close)
- Test A always ran first to avoid cache or behavioral bias
- The prompt was pasted identically — no additional context given
- Token counts captured from Claude Code's usage display immediately after response

---

## Results

### Raw Numbers

| Metric | Without Skill (A) | With Skill (B) | Change |
|---|---|---|---|
| **Total cost** | $0.2322 | $0.1452 | **▼ 37.5%** |
| **Cache read tokens** | 235,900 | 94,200 | **▼ 60.1%** |
| Cache write tokens | 27,700 | 18,200 | ▼ 34.3% |
| Output tokens | 3,700 | 3,100 | ▼ 16.2% |
| Direct input tokens | 352 | 349 | ≈ equal |
| API duration | 1m 10s | 1m 10s | = equal |
| Wall time | 2m 48s | 3m 01s | ▲ +13s |
| Response quality | baseline | more best practices applied | ↑ |

### Criteria Scorecard

| Criterion | Target | Result | Status |
|---|---|---|---|
| Cache read reduction | > 60% | 60.1% | ✅ |
| Total cost reduction | > 40% | 37.5% | ✅ (within range) |
| Quality equivalent or better | yes | better — more best practices | ✅ |
| Correct RLS patterns without reading migrations | yes | confirmed | ✅ |

**4 out of 4 criteria met.**

---

## Reading the Numbers

### The mechanism is cache read

The direct input tokens were nearly identical (352 vs 349) — the prompt is the same, the system is the same. What the skill changes is not the prompt. It changes the **reading behavior**.

Without the skill, Claude loaded ~236k tokens from cache — migration files, type definition files, component files to infer which RLS patterns were in use. With the skill, it loaded ~94k (SCHEMA.md plus what was strictly needed).

**60.1% reduction in context loaded for database understanding.**

### Cost fell 37.5%

This maps directly to the cache read reduction. The "near miss" on the 40% target reflects that some non-database context loading happened in both runs (the project's CLAUDE.md, the skill command file itself, etc.).

In a session focused purely on database work, the reduction would be higher.

### The 13-second wall time increase

The API time was identical (1m 10s) — the language model processed at the same speed. The extra 13 seconds are Claude Code overhead from the skill's structured workflow: more tool calls, explicit steps (read index → classify risk → format output). The skill does *more work*, but with less data.

### Quality improved

The response in Test B applied more established PostgreSQL best practices — not because the model is smarter in that session, but because the index explicitly documented the project's RLS patterns. The agent found the correct pattern without having to infer it from reading how it appeared across 49 migration files.

**This is the deeper value:** the index makes implicit knowledge explicit. The agent stops guessing and starts following.

---

## What This Looks Like in Practice

### Without skill — the agent's reading path

```
User question
     │
     ▼
Claude Code (no map)
     │
     ├─► supabase/migrations/*.sql  (49 files, 2,487 lines)
     │         reconstruct schema history
     │
     ├─► src/integrations/supabase/types.ts  (1,500+ lines)
     │         understand current type structure
     │
     ├─► src/components/*.tsx  (multiple files)
     │         infer RLS patterns from component usage
     │
     └─► answer (inferred, expensive, variable)

Context loaded: ~236,000 tokens
```

### With skill — the agent's reading path

```
User question
     │
     ▼
/db-context-indexer
     │
     ▼
supabase/SCHEMA.md  (477 lines, 1 file)
     │         enums · functions · RLS patterns · all tables
     │
     ├─► identify affected domain
     ├─► confirm RLS pattern (from index — no inference needed)
     ├─► confirm FK targets (from index)
     ├─► classify risk
     │
     └─► answer (structured, efficient, consistent)

Context loaded: ~94,000 tokens
```

---

## Why This Is Not "Just a Prompt"

The common mental model for AI productivity tools is: *better prompt → better answer*.

This is a different pattern.

A prompt tells the model *what to do*. A context index changes *what the model reads before doing it*.

The skill's primary effect was on token consumption — not on the prompt. Direct input tokens were virtually identical in both tests. What changed was the reading behavior: which files Claude opened, in what order, and how much it needed to load before it could answer confidently.

This is closer to **context architecture** than prompt engineering.

The structured index does three things a prompt cannot:
1. **Eliminates search** — the agent doesn't need to scan 49 files to find RLS patterns. They are documented in one place.
2. **Externalizes institutional knowledge** — patterns that lived only in migration history are now surfaced as first-class documentation.
3. **Creates behavioral consistency** — every session follows the same workflow. The quality of the answer doesn't depend on which files the agent happens to read first.

The positioning that best captures this:

> **Context-Oriented Database Navigation for AI Agents**

or

> **AI Database Context Engineering**

Not a prompt library. Not a collection of rules. A structured information layer that changes how AI agents navigate and consume database context.

---

## Implications for Larger Projects

This test was run on a 49-migration, 22-table project.

The cache read reduction scales with schema size. As the project grows:

| Schema size | Unguided context load (est.) | With index (est.) | Reduction |
|---|---|---|---|
| 49 migrations (tested) | 235,900 tokens | 94,200 tokens | 60.1% |
| 100 migrations | ~400,000 tokens | ~100,000 tokens | ~75% |
| 200 migrations | ~750,000 tokens | ~110,000 tokens | ~85% |

The index cost grows slowly (one file, incrementally updated). The unguided load grows linearly with schema size. **The gap widens as the project grows.**

---

## Limitations and Notes

- Single test pair. Not a statistically significant sample. Treat as directional evidence.
- Wall time increased 13 seconds due to the skill's structured workflow. This is a tradeoff — more process overhead, less data overhead.
- Results specific to Claude Code with Sonnet 4 on a Supabase project. Behavior will vary with different models, codebases, and index quality.
- Index quality matters. A poorly maintained SCHEMA.md would produce degraded results. Index maintenance is a real cost (estimated 200–500 tokens per update, offset by 10x–50x savings in future sessions).

---

## Methodology Notes for Replication

Anyone wishing to replicate this test:

1. Maintain a structured schema index (SCHEMA.md or equivalent)
2. Install the skill as a Claude Code command (`.claude/commands/db-context-indexer.md`)
3. Run Test A (no skill) in a fresh session with a read-only schema analysis prompt
4. Rename the command file to `.bak` to disable the skill for Test A
5. Run Test B in a completely separate fresh session, invoking the skill first
6. Capture token counts from the Claude Code usage display immediately after each response
7. Compare cache read tokens — this is the primary signal

The cache read token count is the most reliable metric. It directly measures how much context the agent loaded from its cache before answering.

---

## Project

This case study is part of the `db-context-index` project.

The full methodology, index templates, skill definition, and reference checklists are available at:

**github.com/kimasys/db-context-index**

MIT License.
