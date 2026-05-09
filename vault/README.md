# Vault

This folder tracks project decisions, session logs, and incremental documentation for `db-context-index`.

It is not published documentation — it is a working memory layer for the project.

---

## Structure

```
vault/
├── README.md              ← this file
├── sessions/              ← per-session work logs
│   └── YYYY-MM-DD.md
├── decisions/             ← architectural and design decisions (ADRs)
│   └── NNN-title.md
└── roadmap.md             ← living roadmap and next steps
```

---

## Session Logs

One file per working session, named by date. Each session log captures:

- What was worked on
- Decisions made
- Problems encountered
- Next steps

---

## Decisions

Architectural decisions are logged as lightweight ADRs (Architecture Decision Records):

```
NNN-short-title.md
```

Each decision captures: context, options considered, decision taken, and consequences.

---

## Usage

Start every working session by:

1. Reading the latest session log
2. Checking `roadmap.md` for current priorities
3. Opening the relevant index files in `examples/supabase/db-index/`
