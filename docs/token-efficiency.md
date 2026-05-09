# Token Efficiency

**PT-BR:** Princípios de eficiência de tokens para agentes IA trabalhando com bancos de dados.

---

## The Problem

AI coding agents working with large databases consume tokens inefficiently by default.

A typical unguided session might include:

- Reading 5–20 migration files to understand what already exists
- Scanning full schema dumps to find one table definition
- Re-reading the same RLS policies multiple times across turns
- Loading historical context that has no relevance to the current task

For a mid-size Supabase project (50 tables, 200 migrations, full RLS), an unguided agent may consume 30,000–80,000 tokens just loading context before writing a single line of SQL.

**db-context-index cuts this down to 3,000–8,000 tokens for the same task.**

---

## Why Index-First Works

A database index does not replace the schema — it maps it.

Like a book index, it lets you jump directly to what you need without reading every page.

The agent's token budget is spent on:

1. Reading the index (small — fixed cost regardless of schema size)
2. Reading only the SQL files identified by the index (targeted — proportional to scope)
3. Generating the migration (the actual work)

Without the index, step 1 and 2 collapse into an expensive scan of everything.

---

## Token Budget Guidelines

| Task Type | Without Index | With Index | Reduction |
|---|---|---|---|
| Add nullable column | 15k–30k tokens | 2k–4k tokens | ~85% |
| Modify RLS policy | 30k–60k tokens | 4k–8k tokens | ~87% |
| Add foreign key | 20k–40k tokens | 3k–6k tokens | ~85% |
| New table with RLS | 10k–20k tokens | 2k–4k tokens | ~80% |
| Complex trigger change | 40k–80k tokens | 6k–12k tokens | ~85% |

*Estimates based on a 50-table Supabase project with full RLS. Actual savings scale with schema size.*

---

## Context Loading Rules

### Load in order — stop when you have enough

```
1. 00-overview.md       → always (cheap, always useful)
2. relevant domain file  → when domain is identified
3. 02-rls-policies.md   → only if RLS is affected
4. 03-functions.md      → only if functions/triggers are affected
5. 04-relationships.md  → only if schema structure changes
6. specific SQL files    → only the ones identified above
```

Never load files 2–6 by default. Always derive from the overview first.

---

## Anti-Patterns to Avoid

### Reading the full migration history

```
❌ Read all 200 migration files to understand the current schema
✓  Read 00-overview.md and 01-tables.md to get the current state
```

The index represents current state. Historical migrations are only relevant when debugging a specific historical change.

---

### Loading unrelated domains

```
❌ Load all RLS policies to work on a task-domain change
✓  Load only the policies for tables in the affected domain
```

Each domain in 02-rls-policies.md is clearly labeled. Scope your reads.

---

### Re-reading context across turns

```
❌ Re-read schema files at the start of each turn
✓  Summarize what was loaded, carry forward only the relevant pieces
```

An agent should maintain a working memory of what it loaded and avoid redundant reads within a session.

---

### Scanning for dependencies instead of using the index

```
❌ grep through all SQL files to find what references a table
✓  read 04-relationships.md which maps all FK dependencies
```

---

## The Compounding Benefit

Token efficiency compounds across a project lifetime.

- Session 1: Agent loads the index once, executes migration
- Session 2: Agent loads the index (already compact), continues work
- Session N: Index grows incrementally, but remains navigable because it is structured

Without the index, every session restarts from scratch — re-loading the same context at full cost.

---

## Index Maintenance Cost

Keeping the index accurate has a cost.

After every migration:
- Update `01-tables.md` if tables changed
- Update `02-rls-policies.md` if policies changed
- Update `03-functions-triggers.md` if functions or triggers changed
- Update `04-relationships.md` if FK relationships changed
- Update `99-change-log.md` always

This maintenance cost is typically **200–500 tokens** of index updates per migration, compared to **10,000–50,000 tokens** saved in future sessions that use the updated index.

**Maintenance ROI: 20x–100x.**
