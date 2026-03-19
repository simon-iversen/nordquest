# Nordquest — Root Orchestration Context

This directory contains all nordquest repositories. Claude auto-loads this file whenever running from any sub-repo.

## Repos

| Repo | Purpose | Orchestrated |
|------|---------|-------------|
| `nordquest_app/` | Flutter mobile app (primary working repo) | Yes |
| `nordquest_supabase/` | Supabase backend: migrations, RLS, edge functions | Yes — use `supabase-agent` |
| `nordquest_worker/` | Python background workers on Railway | Yes — use `worker-agent` |
| `nordquest_homepage/` | Marketing homepage | Out of scope |
| `nordquest_tools/` | Internal tooling | Yes — use `tools-agent` |

## Cross-Repo Dependency Map

Changes flow in this order:

```
nordquest_supabase/ (DB schema, RPC, RLS)
        ↓
nordquest_worker/ (reads/writes DB via RPC)
        ↓
nordquest_app/ (reads DB, calls edge functions)
```

When making changes that span repos, always start from the bottom of the stack (Supabase) and work upward.

## Agent Routing

- **Database tasks** (tables, indexes, RLS, RPC functions, migrations, edge functions): delegate to `supabase-agent`
- **Worker tasks** (job processing, Strava sync, place matching, Railway deployment): delegate to `worker-agent`
- **Flutter tasks**: handle directly in `nordquest_app/`
- **Tools tasks** (data generation scripts, trail/shelter ingestion, OSM processing, media uploads): delegate to `tools-agent`

## Agent Definitions

- nordquest-tools-agent: Handles all data generation, trail/shelter ingestion, OSM processing, and media asset management. Invoke when a task involves running or modifying scripts in nordquest_tools. Cross-delegates to supabase-agent for schema questions. Works in /Users/simoniversen/Development/nordquest/nordquest_tools/.

## After Any Supabase Schema Change

After modifying any schema or RPC function in `nordquest_supabase/`, remind the user to run:

```bash
python3 tools/generate_backend_docs.py
```

from the `nordquest_supabase/` directory. This regenerates docs in `docs/backend/generated/`.

## Key Cross-Repo Contracts

- The RPC functions `update_pending_matching_job_to_running` and `update_pending_job_to_running` are called by the workers to claim jobs. Never change their signatures without coordinating changes in both `nordquest_supabase/` (migration) and `nordquest_worker/` (code).
