# FeedEater - Project Catalog

**Repo**: https://github.com/jhacksman/FeedEater
**Last commit**: 2026-02-04 (29 commits, all AI-generated via GPT-5.2 + Cursor)
**Stack**: TypeScript (strict) monorepo, Node >= 20, ESM throughout

---

## What It Is

Self-hosted newsfeed engine with a modular plugin architecture. Collects content from social platforms, RSS feeds, prediction markets, and messaging apps into a unified message bus. AI-generated summaries and semantic embeddings provide "context" — evolving conversation summaries that grow as new messages arrive.

Philosophy: "Contexts allow deep knowledge." Instead of just aggregating links, FeedEater builds living summaries of conversations and topics across all sources.

---

## Architecture Overview

```
                         port 666
                       +---------+
                       |  Caddy  |  (reverse proxy)
                       +----+----+
                            |
              +-------------+-------------+
              |                           |
        +-----+------+           +-------+------+
        | Next.js Web |           |  Express API |
        |  (port 3000)|           |  (port 4000) |
        +-------------+           +------+-------+
                                         |
                    +--------------------+--------------------+
                    |                    |                    |
              +-----+-----+      +------+------+      +-----+-----+
              |   Worker   |      |  Postgres   |      |    NATS   |
              | (job exec) |      |  (pgvector) |      | JetStream |
              +------------+      +-------------+      +-----------+
```

### Services (docker-compose)

| Service  | Image / Build            | Purpose                              |
|----------|--------------------------|--------------------------------------|
| postgres | pgvector/pgvector:pg16   | Storage + vector search              |
| nats     | nats:2.10-alpine         | Event bus (JetStream enabled)        |
| api      | Dockerfile.api           | REST API + SSE streams               |
| worker   | Dockerfile.worker        | Job execution + bus archiver         |
| web      | Dockerfile.web           | Next.js dashboard                    |
| proxy    | caddy:2-alpine           | Reverse proxy on port 666            |

### Data Flow

1. **Module cron fires** (e.g., every 5 min for Slack `collect`)
2. Worker publishes `JobRun` event to NATS
3. Worker dispatches to module's job handler
4. Handler calls platform API (Slack, Reddit, etc.) and stores raw data in module's private Postgres schema (`mod_<name>`)
5. Handler publishes `MessageCreated` event to `feedeater.<module>.messageCreated`
6. Bus archiver persists to `bus_messages` table
7. SSE endpoint streams to web dashboard in real time
8. Every 30 min, `updateContexts` job generates AI summaries via Ollama, publishes `ContextUpdated`
9. Worker upserts context summaries + embeddings into `bus_contexts`

---

## Monorepo Structure

### Apps

| App    | Package              | Framework    | Lines (index) | Status     |
|--------|----------------------|-------------|---------------|------------|
| api    | @feedeater/api       | Express 4   | 154           | Functional |
| web    | @feedeater/web       | Next.js 14  | 10 (page)     | Functional |
| worker | @feedeater/worker    | Node (raw)  | 679           | Functional |

**API endpoints** (apps/api/src/index.ts):
- `GET /api/health` - Health check
- `GET /api/modules` - List discovered modules
- `GET /api/bus/stream` - SSE live message stream
- `GET /api/bus/history` - Archived messages from Postgres
- `GET /api/contexts/history` - Context summaries
- `GET /api/contexts/messages` - Messages linked to contexts
- `GET /api/contexts/stream` - SSE context updates
- `GET /api/settings/:module` - Module settings (encrypted secrets masked)
- `PUT /api/settings/:module/:key` - Update a setting
- `GET /api/internal/settings/:module` - Internal (decrypted) settings for worker
- `POST /api/internal/ai/summary` - Run Ollama summary
- `POST /api/internal/ai/embedding` - Generate embedding vector
- `GET /api/internal/ai/tags` - AI tag suggestions
- `GET /api/modules/slack/channels` - Slack channel list
- `GET /api/logs/stream` - SSE log stream
- `GET /api/jobs/status` - Job status overview
- `POST /api/jobs/run` - Manual job trigger

**Web UI pages**:
- `/` - Dashboard (LiveBusFeed component)
- `/modules` - Module index + per-module settings/cards
- `/jobs` - Job status + manual run panel
- `/contexts` - Context summary feed
- `/logs` - Live log stream

**Worker capabilities**:
- Module discovery from filesystem
- Dynamic runtime loading (`dist/runtime.js`)
- Cron scheduling (lightweight, no external scheduler)
- NATS-triggered job dispatch
- Bus archiver (JetStream -> Postgres)
- Context upsert with pgvector embeddings
- Startup re-emit (replays recent Postgres messages to NATS)

### Packages

| Package    | Package Name          | Purpose                            | Key Exports                          |
|------------|----------------------|-------------------------------------|--------------------------------------|
| core       | @feedeater/core      | Shared contracts + utilities        | Zod schemas, NATS subject helpers, settings client |
| db         | @feedeater/db        | Prisma client                       | `prisma` singleton                   |
| module-sdk | @feedeater/module-sdk| Module runtime interface            | `ModuleRuntime`, `ModuleRuntimeContext`, `ModuleJobHandler` |

**Core contracts** (Zod schemas):
- `NormalizedMessageSchema` - Canonical message shape (id, createdAt, source, Message, tags, contextRef, followMePanel, etc.)
- `MessageCreatedEventSchema` - Bus event wrapping a NormalizedMessage
- `ContextUpdatedEventSchema` - Context summary update event
- `TagAppendedEventSchema` - Tag enrichment event
- `JobRunEventSchema` - Job trigger event
- `JobTriggerSchema` - Trigger metadata (schedule/manual/event)

**Prisma schema** (7 models):
- `Setting` - Encrypted key/value config per module
- `BusMessage` - Archived normalized messages
- `BusTag` - Tag enrichments on messages
- `BusContext` - Context summaries with pgvector embeddings
- `BusContextMessage` - Message-context associations
- `JobRun` - Per-execution job records
- `JobState` - Last-run state per module/job
- `BusReemitDedupe` - Startup re-emit deduplication

---

## Modules

All modules follow the same pattern: `module.json` manifest + `settings.ts` types + `src/ingest.ts` (core logic) + `src/runtime.ts` (job handlers).

### Implementation Status

| Module      | ingest.ts Lines | Has CLAUDE.md | Has SQL dir | Jobs                        | Schedule        | Status       |
|-------------|----------------|---------------|-------------|-----------------------------|-----------------|--------------| 
| slack       | 688            | Yes           | Yes         | collect, updateContexts     | 5m, 30m         | Implemented  |
| rss         | 1234           | Yes (488 LOC) | No          | poll, cleanup, updateContexts | 5m, daily, 30m | Implemented  |
| reddit      | 825            | Yes           | No          | collect, updateContexts     | 10m, 30m        | Implemented  |
| discord     | 933            | Yes           | No          | collect, updateContexts     | 5m, 30m         | Implemented  |
| telegram    | 1076           | Yes           | No          | collect, updateContexts     | 5m, 30m         | Implemented  |
| twitter     | 986            | Yes           | No          | collect, updateContexts     | 15m, 30m        | Implemented  |
| youtube     | 801            | Yes           | Yes         | collect, updateContexts     | 1h, 6h          | Implemented  |
| bluesky     | 773            | Yes           | No          | collect, updateContexts     | 5m, 30m         | Implemented  |
| mastodon    | 797            | Yes           | No          | collect, updateContexts     | 5m, 30m         | Implemented  |
| signal      | 1130           | Yes           | Yes         | collect, sessionCheck, updateContexts | 1m, 6h, 30m | Implemented |
| twitch      | 1270           | Yes           | No          | collect, updateContexts     | 5m, 30m         | Implemented  |
| polymarket  | 273            | No            | No          | collect, updateContexts     | 5m, 30m         | Stub         |
| kalshi      | 257            | No            | No          | collect, updateContexts     | 5m, 30m         | Stub         |
| example     | N/A            | No            | No          | tick, processMessage        | 1m, event       | Reference    |
| system      | N/A            | No            | No          | ollamaDebug                 | manual          | Utility      |

### Module Auth Methods

| Module     | Auth Type                          | Required Settings                                  |
|------------|------------------------------------|----------------------------------------------------|
| slack      | Bot token (xoxb-)                  | botToken, channelIds                               |
| rss        | None (public feeds)                | Feed URLs via UI                                   |
| reddit     | OAuth2 (script app)                | clientId, clientSecret, username, password          |
| discord    | Bot token                          | botToken, guildIds                                 |
| telegram   | MTProto (user session)             | apiId, apiHash, sessionString                      |
| twitter    | Browser cookies                    | cookieSource (chrome/firefox profile)              |
| youtube    | RSS (free) or OAuth2               | channelIds; optional oauthCredentials, apiKey       |
| bluesky    | App password (AT Protocol)         | identifier, appPassword                            |
| mastodon   | OAuth access token                 | instanceUrl, accessToken                           |
| signal     | signal-cli daemon                  | signalCliUrl, phoneNumber                          |
| twitch     | OAuth2 (app + user token)          | clientId, clientSecret, userAccessToken, userId    |
| polymarket | None (public API)                  | watchedMarkets, watchedCategories                  |
| kalshi     | API key (optional for public)      | apiKey, apiSecret (optional)                       |

---

## Non-Negotiable Rules (from agentinstructions.md)

1. Do not modify `agentinstructions.md`
2. Always re-read `agentinstructions.md` before making changes
3. No secrets in git: all secrets/config via `.env` and module settings
4. Docker-first: everything must run via `docker-compose`
5. Re-deployable: changes must be safe to apply repeatedly (idempotent deploy + migrations)
6. TypeScript strict: keep strict mode enabled; avoid `any`
7. Use NATS: Do not re-introduce BullMQ/Redis/BullBoard
8. Workers are cattle: All important data must be replicated to Postgres storage layer
9. Fail fast: unexpected states should throw with clear logs
10. Settings over Constants: use module settings with default values over hard-coded constants
11. Use the glossary: Always use terminology from GLOSSARY.md

### Interop Contract

- No cross-module database access (each module owns `mod_<name>` schema)
- No direct module-to-module imports/calls
- All cross-module communication via NATS events and/or tags
- Modules build prompts and parse results; platform owns LLM execution

---

## What's Working

- Full platform skeleton: API + Worker + Web UI + Postgres + NATS + Caddy
- Module discovery + dynamic runtime loading
- Encrypted-at-rest settings registry
- Bus archiver (NATS -> Postgres)
- SSE live streams (messages, contexts, logs)
- Cron scheduler + manual job triggers
- AI context summarization pipeline (via Ollama)
- Semantic embeddings with pgvector
- 11 fully implemented source modules (social + messaging)
- 2 prediction market stubs
- Dashboard with live feed, module management, job status, context viewer, log viewer

## What's Missing / Incomplete

### Platform Level
- No tests (zero test files anywhere)
- No linting configured (api + worker just `echo "(no lint configured yet)"`)
- No CI/CD pipeline
- No authentication on the web UI or API
- No rate limiting on API endpoints
- Worker has `as any` casts in several places (module-sdk type gaps)
- No health checks for worker process
- No graceful shutdown handling
- Changelog only has 1 entry (initial skeleton)

### Module Level
- Polymarket and Kalshi are stubs (273/257 lines vs 700-1200 for real modules)
- Only Slack has a custom UI card component (`SlackChannelsCard.tsx`); other modules rely on generic settings editor
- No OPML import/export for RSS
- No feed discovery for RSS (paste URL only)
- Twitter relies on browser cookie scraping (fragile)
- Signal requires external `signal-cli` daemon
- No module-level tests
- Worker only loads `example` and `slack` in its package.json dependencies (other modules discovered from filesystem but may not be in the Docker build)

### Documentation
- ARCHITECTURE.md is a skeleton (20 lines)
- No deployment guide beyond docker-compose
- No user-facing documentation
- CLAUDE.md files exist for 11 modules but not for polymarket, kalshi, example, or system

---

## Patterns and Conventions

### Module Anatomy
```
modules/<name>/
  module.json        # Manifest: name, jobs, settings, cards
  settings.ts        # Zod types mirroring module.json settings
  package.json       # Dependencies (always @feedeater/core + @feedeater/module-sdk)
  tsconfig.json      # Extends base TS config
  src/
    index.ts         # Re-exports
    ingest.ts        # Core logic class (<Name>Ingestor)
    runtime.ts       # createModuleRuntime() -> ModuleRuntime with job handlers
  CLAUDE.md          # Boris-method knowledge file (lessons learned)
  README.md          # Brief description
  PLAN.md            # Implementation plan (some modules)
```

### Ingestor Pattern
Every module's `ingest.ts` follows:
1. `ensureSchema()` - Create private Postgres tables if not exists
2. `collectAndPersist()` / `pollFeeds()` - Fetch from external API, dedupe, store, publish MessageCreated
3. `refreshContexts()` - Generate AI summaries, publish ContextUpdated
4. Internal helper methods for API calls, deduplication, embedding generation

### NATS Subject Convention
- `feedeater.<module>.messageCreated` - New message
- `feedeater.<module>.contextUpdated` - Context summary update
- `feedeater.<module>.log` - Module logs
- `feedeater.tags.appended` - Tag enrichment
- `feedeater.jobs.<module>.<queue>.<job>` - Job triggers
- `feedeater.worker.log` - Worker-level logs

### Settings Flow
1. User configures via web UI (`PUT /api/settings/:module/:key`)
2. Stored encrypted in Postgres `Setting` table
3. Worker fetches decrypted via `GET /api/internal/settings/:module` (bearer token auth)
4. Module parses with its typed settings parser

---

## Environment

Required env vars:
- `DATABASE_URL` - Postgres connection string
- `NATS_URL` - NATS connection string
- `FEED_SETTINGS_KEY` - 32-byte base64 key for encrypting secrets at rest
- `FEED_INTERNAL_TOKEN` - Bearer token for worker -> API internal calls

Optional:
- `OLLAMA_BASE_URL` - Ollama endpoint for AI features
- `OLLAMA_SUMMARY_MODEL` / `OLLAMA_EMBED_MODEL` - Model names (default: llama3.1:8b)
- `OLLAMA_EMBED_DIM` - Embedding vector size (default: 4096)

---

## Build System

- Monorepo via npm workspaces (`apps/*`, `packages/*`, `modules/*`)
- Build: `tsup` for API/worker/packages, `next build` for web
- TypeScript strict mode everywhere
- No bundler for modules (tsup to ESM)
- Prisma for schema management (`prisma db push`)
- Docker multi-stage builds for each service

---

*Cataloged 2026-02-11 by Devin. Following the [Boris Cherny Method](https://github.com/jhacksman/boris-method).*
