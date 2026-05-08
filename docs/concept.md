# Concept

## Overview

`db-context-index` is a structured methodology for helping AI coding agents work safely and efficiently with large PostgreSQL and Supabase schemas.

Instead of forcing the agent to read the entire database before every migration, the system creates indexed database context files.

These files act as a lightweight navigation layer.

---

## Main problem

Large SQL schemas create three major problems for AI agents:

1. Excessive token usage
2. Slow context loading
3. Increased migration risk

As projects grow, agents start reading:

- tables
- policies
- triggers
- functions
- views
- indexes
- relationships
- historical migrations

Even when only a small part of the schema is relevant.

---

## Proposed solution

The solution is to split database understanding into indexed domains.

Instead of reading the entire schema:

1. Read compact index files first
2. Identify the affected domain
3. Load only the required SQL context
4. Execute migration planning
5. Update the index after changes

---

## Philosophy

The goal is not only token reduction.

The goal is:

- safer migrations
- deterministic context loading
- reusable database knowledge
- better AI reasoning
- reduced hallucination risk
- lower cognitive overload

---

## Human + AI alignment

One important principle:

The same database map should help both humans and AI agents.

The index becomes:

- onboarding documentation
- migration memory
- dependency map
- architectural reference
- AI navigation layer

---

## Future directions

Potential future directions include:

- automatic schema indexing
- dependency graph generation
- semantic retrieval
- migration risk analysis
- AI-aware database linting
- context compression
- schema clustering
- vector retrieval
- IDE integrations
