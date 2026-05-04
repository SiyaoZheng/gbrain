# CLAUDE.md

GBrain is a personal knowledge brain and GStack mod for agent platforms. Pluggable
engines: PGLite (embedded Postgres via WASM, zero-config default) or Postgres + pgvector
+ hybrid search (Supabase, Aliyun RDS, or self-hosted). `gbrain init` defaults to PGLite;
suggests Postgres for 1000+ files. Production brain runs on Aliyun RDS PostgreSQL
(cn-beijing, pgm-2ze4oes6klt9y802) via SSH tunnel through ECS jump host.

## Commands

Quick reference for the most-used commands:

- `gbrain init` — Initialize a brain (defaults to PGLite)
- `gbrain sync [--skip-failed] [--retry-failed] [--workers N]` — Sync markdown files into the brain
- `gbrain doctor [--json] [--fast] [--fix] [--dry-run]` — Health checks
- `gbrain embed [--stale|--all] [--slugs ...]` — Generate embeddings
- `gbrain extract links|timeline|all [--source fs|db]` — Batch link/timeline extraction
- `gbrain graph-query <slug> [--type T] [--depth N] [--direction in|out|both]` — Relationship traversal
- `gbrain dream [--dir P] [--dry-run] [--phase N] [--input F] [--date D]` — Run maintenance cycle
- `gbrain agent run <prompt> [--follow]` — Submit a subagent job
- `gbrain jobs submit <name> [--params JSON] [--follow]` — Submit a background job
- `gbrain jobs list/get/cancel/retry/delete/prune/stats` — Job management
- `gbrain jobs work [--queue Q] [--concurrency N]` — Start worker daemon (Postgres only)
- `gbrain migrate --to postgres` — Migrate PGLite → Postgres (RDS/Supabase/self-hosted)
- `gbrain serve --http` — HTTP transport (Postgres-only)
- `gbrain auth create/list/revoke/test` — Token management for HTTP transport
- `gbrain skillify scaffold <name>` — Create skill stubs
- `gbrain skillpack install` — Install curated skill bundle into host workspace
- `gbrain claw-test [--scenario N] [--live --agent openclaw]` — End-to-end friction harness
- `gbrain friction log|render|list|summary` — Friction/delight reporter

Run `gbrain --help` for full reference.

### Skills

Skills live in `skills/`, organized by `skills/RESOLVER.md` (also accepts `AGENTS.md`).
Key skills: ingest, query, maintain, enrich, briefing, migrate, setup, publish,
signal-detector, brain-ops, meeting-ingestion, minion-orchestrator, and more.
Cross-cutting conventions in `skills/conventions/`.

## Skill routing

When the user's request matches an available skill, invoke it using the Skill tool as your FIRST action.
Key routing rules:
- Ship, deploy, push, create PR → invoke ship
- Bugs, errors, "why is this broken" → invoke investigate
- Code review → invoke review
- QA, test the site → invoke qa
- Update docs after shipping → invoke document-release
- Architecture review → invoke plan-eng-review
- Save progress → invoke checkpoint

## Build

```bash
bun build --compile --outfile bin/gbrain src/cli.ts
```

## Testing

`bun test` runs all unit tests (no database required).

`bun run test:e2e` runs E2E tests against real Postgres+pgvector. Requires `DATABASE_URL`.

### API keys

Always source the user's shell profile before running tests:

```bash
source ~/.zshrc 2>/dev/null || true
```

This loads `OPENAI_API_KEY` and `ANTHROPIC_API_KEY`. Without these, Tier 2 tests skip silently.

### Embedding config

Local llama-server (Qwen3-Embedding-0.6B Q4_K_M, 1024d) — no cloud API needed.

| Component | Detail |
|-----------|--------|
| Model | `~/.cache/llama-models/Qwen3-Embedding-0.6B.Q4_K_M.gguf` (385MB) |
| Server | `llama-server --embeddings` on `localhost:8080` |
| Launch | `launchctl load ~/Library/LaunchAgents/com.gbrain.llama-embed.plist` |
| Speed | ~260 embeddings/s on MPS (0.038s for 10 × 8192 ctx) |

`embedding.ts` reads `GBRAIN_EMBEDDING_BASE_URL` (default `http://localhost:8080/v1`) and `GBRAIN_EMBEDDING_API_KEY` (default `local`). No cloud dependency.

### E2E test DB lifecycle

1. **Check for `.env.testing`** — if missing, copy from a sibling worktree. Read it to get the DATABASE_URL (it has the port number).
2. **Check if the port is free:**
   `docker ps --filter "publish=PORT"` — if another container is on that port, pick a different one (try 5435, 5436, 5437).
3. **Start the test DB:**
   ```bash
   docker run -d --name gbrain-test-pg \
     -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres \
     -e POSTGRES_DB=gbrain_test \
     -p PORT:5432 pgvector/pgvector:pg16
   ```
   Wait for ready: `docker exec gbrain-test-pg pg_isready -U postgres`
4. **Run E2E tests:**
   `DATABASE_URL=postgresql://postgres:postgres@localhost:PORT/gbrain_test bun run test:e2e`
5. **Tear down immediately after tests finish (pass or fail):**
   `docker stop gbrain-test-pg && docker rm gbrain-test-pg`

Never leave `gbrain-test-pg` running. If you find a stale one from a previous run, stop and remove it before starting a new one.