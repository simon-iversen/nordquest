# Nordquest — Root Orchestration Context

This directory contains all nordquest repositories. Claude auto-loads this file whenever running from any sub-repo.

## Product Vision

Nordquest is a **gamified outdoor exploration app for Norway** — think a nature-focused, Nordic take on Pokemon Go. The core loop: users connect their GPS watch (Strava now, Garmin coming soon), go outdoors, and automatically collect places they visit — DNT cabins, shelters, trails, and more.

### Direction & Identity

- **Nature, wilderness, and gamification** — this is the core identity. No urban features.
- **Gamified but tasteful** — more engagement than traditional Norwegian nature apps (which tend to be understated and utilitarian), but not as aggressive as Duolingo. The name "Nordquest" is intentionally international, not a Norwegian-language name.
- **Differentiation** — Norwegian nature apps are too "down to earth." Nordquest brings energy, progression, and motivation to outdoor exploration while respecting the outdoor culture.

### Current Features

- Strava integration for automatic place/trail matching
- DNT cabins, shelters, nature attractions
- Trail progress tracking
- Municipality exploration progress
- Activity history with segment matching

### Planned Features & Expansion

- **Manual GPS check-in** — tap a button in the app to check in at a place (e.g. a mountain summit) without needing a recorded activity. Useful for quick visits or when not tracking.
- **In-app GPS tracking** — built-in route recording for users who don't have a GPS watch. Lowers the barrier for older users or anyone without a Strava/Garmin device.
- **Garmin integration** — coming soon; less restrictive data access than Strava, enabling richer features (public leaderboards, competitions, passive tracking with just a watch)
- **Mountains** — peak bagging, summit tracking
- **National parks** — visit tracking and progress
- **Beaches** — coastal exploration
- **Islands** — island hopping progress
- **Winter/ski peaks** — typical peaks for ski touring
- **Quests system** — user-created and curated quest lists (e.g. "Top 10 mountains in Lofoten", "All national parks", "70% of trail X"). Famous outdoors people / influencers could create signature quests.
- **Leaderboards & competitions** — public rankings, challenges
- **Tourist targeting** — expand audience beyond Norwegian residents to tourists visiting Norway

### Design Principles for Features

- Every new place type or feature should reinforce the collect-explore-progress loop
- Gamification should motivate people to get outdoors, not feel like a chore
- Passive data collection (GPS watch) is the ideal UX — minimal manual input

## Repos

| Repo | Purpose | Orchestrated |
|------|---------|-------------|
| `nordquest_app/` | Flutter mobile app (primary working repo) | Yes |
| `nordquest_supabase/` | Supabase backend: migrations, RLS, edge functions | Yes — use `supabase-agent` |
| `nordquest_worker/` | Python background workers on Railway | Yes — use `worker-agent` |
| `nordquest_homepage/` | Marketing homepage | Out of scope |
| `nordquest_dashboard/` | Worker monitoring dashboard (static HTML) | Out of scope |
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

- supabase-agent: Handles database schema changes, migrations, RPC functions, RLS policies, and edge functions. Works in /Users/simoniversen/Development/nordquest/nordquest_supabase/. Creates new migration files (never edits existing ones).
- worker-agent: Handles Python background worker changes (sync and matching). Works in /Users/simoniversen/Development/nordquest/nordquest_worker/. Cross-delegates to supabase-agent for schema changes.
- nordquest-tools-agent: Handles all data generation, trail/shelter ingestion, OSM processing, and media asset management. Invoke when a task involves running or modifying scripts in nordquest_tools. Cross-delegates to supabase-agent for schema questions. Works in /Users/simoniversen/Development/nordquest/nordquest_tools/.

## After Any Supabase Schema Change

After modifying any schema or RPC function in `nordquest_supabase/`, remind the user to run:

```bash
python3 tools/generate_backend_docs.py
```

from the `nordquest_supabase/` directory. This regenerates docs in `docs/backend/generated/`.

## Umbrella Repo Sync

This directory is an umbrella git repo with all sub-repos as git submodules. After pushing commits in any sub-repo, update the umbrella so Claude Code on the web sees the latest:

```bash
cd /Users/simoniversen/Development/nordquest
git add nordquest_app nordquest_supabase nordquest_tools nordquest_worker nordquest_homepage
git commit -m "Update submodule pointers"
git push
```

Do this automatically after any `git push` in a sub-repo — no need to ask the user.

## Key Cross-Repo Contracts

- The RPC functions `update_pending_matching_job_to_running` and `update_pending_job_to_running` are called by the workers to claim jobs. Never change their signatures without coordinating changes in both `nordquest_supabase/` (migration) and `nordquest_worker/` (code).
