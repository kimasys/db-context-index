# Database Overview

## Purpose

This file provides a high-level map of the database.

The goal is to help AI agents understand the database structure before opening raw SQL files.

---

## Main domains

| Domain | Description |
|---|---|
| authentication | users, sessions, auth-related entities |
| organization | tenants, teams, permissions |
| operations | projects, tasks, workflows |
| analytics | dashboards, metrics, reports |
| system | logs, events, internal automation |

---

## Critical systems

- RLS policies
- tenant isolation
- ownership rules
- audit logging
- auth relationships

---

## Important dependencies

- authentication affects all tenant-scoped entities
- tenant_id relationships are critical for isolation
- policy helper functions may affect multiple tables

---

## Recommended behavior for AI agents

Before reading SQL:

1. Identify affected domain
2. Inspect dependency impact
3. Inspect RLS impact
4. Open only required SQL files
