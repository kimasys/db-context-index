# Roadmap

**PT-BR:** Roteiro de desenvolvimento do projeto.

---

## Vision

`db-context-index` aims to become the standard pattern for AI-assisted database engineering with PostgreSQL and Supabase.

The end state: any developer can drop a `db-index/` folder into their project and immediately give any AI agent a safe, efficient, structured view of their database.

---

## v1 — Documentation-First (current)

**Goal:** A complete, usable methodology that works without any automation. Humans fill the index manually or with AI assistance.

### Deliverables

- [x] Core concept documentation (`docs/concept.md`)
- [x] Methodology documentation (`docs/methodology.md`)
- [x] Token efficiency analysis (`docs/token-efficiency.md`)
- [x] Complete 7-file index example for Supabase (`examples/supabase/db-index/`)
- [x] Claude Code skill (`skills/db-context-indexer/SKILL.md`)
- [x] Migration safety checklist (`skills/db-context-indexer/references/`)
- [x] PostgreSQL best practices reference (`skills/db-context-indexer/references/`)
- [x] Supabase RLS checklist (`skills/db-context-indexer/references/`)
- [x] Real-world benchmark and case study (`docs/case-study-claude-code.md`)

### Success criteria — MET ✅

Tested on a real Supabase production project (49 migrations, 22 tables):
- Cache read tokens: ▼ 60.1% (235,900 → 94,200)
- Total cost: ▼ 37.5% ($0.2322 → $0.1452)
- Response quality: improved (more best practices applied)
- 4/4 validation criteria passed

---

## v2 — Automation Scripts

**Goal:** CLI scripts that generate and update index files from a live database.

### Planned deliverables

- [ ] `scripts/generate-index.sql` — query a live Supabase database and output all 7 index files
- [ ] `scripts/update-change-log.sql` — append a migration entry to `99-change-log.md`
- [ ] `scripts/validate-index.sql` — detect drift between the index and the live schema
- [ ] Node.js or Python CLI wrapper (TBD based on community feedback)

### Notes

Automation requires live database access. Security of credentials and read-only access patterns must be addressed.

---

## v3 — Intelligent Context

**Goal:** Semantic and graph-based retrieval to make context loading even more precise.

### Planned directions

- [ ] Dependency graph generation (visualize table relationships as a DAG)
- [ ] Semantic search over index files (find relevant context by natural language)
- [ ] Migration risk analysis automation (infer risk from schema diff)
- [ ] AI-aware database linting (flag anti-patterns before migration)
- [ ] Vector retrieval integration (embed index files for fast similarity search)
- [ ] Context compression (reduce index size for very large schemas)

---

## Skills Architecture — Broader Vision

`db-context-index` is designed as the first skill in a larger architecture.

The goal is a suite of Claude Code skills covering every domain of a large digital platform project:

| Domain | Skill | Status |
|---|---|---|
| Database | `db-context-indexer` | v1 in progress |
| Backend | `api-context-indexer` | planned |
| Authentication | `auth-context-indexer` | planned |
| Frontend | `ui-context-indexer` | planned |
| Infrastructure | `infra-context-indexer` | planned |
| Testing | `test-context-indexer` | planned |

Each skill follows the same pattern:
- Structured index files for its domain
- Token-efficient context loading rules
- Risk classification for changes
- Workflow guidance for the AI agent

---

## Contributing

See `CONTRIBUTING.md` for how to contribute to any phase of the roadmap.

Community feedback on v1 will shape v2 priorities.
