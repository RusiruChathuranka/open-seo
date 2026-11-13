# Vercel Migration Plan

## Target

Run the OpenSEO application on Vercel (Node.js runtime) with Neon PostgreSQL. Preserve the existing product surface and behavior; do not remove features to make the first deployment easier.

## Current infrastructure

- Frontend/server rendering: TanStack Start + React 19 + Vite.
- Database: Drizzle ORM with PostgreSQL schema and migration history already present.
- Current hosting/runtime: Cloudflare Workers with Cloudflare Vite plugin.
- Cloudflare-specific capabilities in use:
  - Hyperdrive-backed PostgreSQL connection and `cloudflare:workers` environment access.
  - R2 object storage for audit payloads.
  - Durable Workflows for site audits, rank checks, AI visibility, and local rank checks.
  - Durable Objects / Agents SDK for persistent SAM chat sessions.
  - Workers OAuth provider and KV-backed MCP OAuth registrations, grants, and tokens.
  - Worker scheduled handler for maintenance, reconciliation, and recurring rank checks.

## Migration principles

1. Keep `main` unchanged. All migration work belongs on `migration/vercel-encore` until reviewed.
2. Keep the existing PostgreSQL schema and migration history; do not apply migrations until the Vercel runtime is ready and a migration dry-run is reviewed.
3. Preserve tenant isolation, authorization checks, billing holds/settlement, idempotency, retry behavior, and audit/run reconciliation.
4. Do not replace durable workflows with fire-and-forget serverless requests.
5. Do not deploy production until the production build, tests, database migration, auth, MCP, SAM streaming, and background jobs have been verified.
6. Do not add paid third-party services or provision billable resources without explicit approval.

## Workstreams

### 1. Vercel application runtime

- Remove the Cloudflare Vite plugin from the production Vercel build path.
- Keep TanStack Start SSR and route behavior on the Node.js runtime.
- Replace Worker `fetch(request, env, ctx)` orchestration with supported TanStack Start server routes / Node handlers.
- Move scheduled work to Vercel Cron endpoints with authenticated invocations.
- Confirm websocket/streaming behavior for SAM and MCP under Vercel Functions.

### 2. Database

- Use `POSTGRES_DATABASE_URL` from Vercel environment variables.
- Replace `cloudflare:workers` / Hyperdrive lookup with Node process environment access.
- Use a reusable Node PostgreSQL pool with bounded connections and correct serverless lifecycle behavior.
- Retain Drizzle PostgreSQL schema and migrations.
- Run migrations only after a preview deployment and schema compatibility review.

### 3. Object storage

- Neon Object Storage is already enabled on the dedicated `open-seo` database branch and exposes an S3-compatible endpoint in the same AWS Singapore region.
- Evaluate Neon Object Storage as the first-choice R2 replacement to avoid introducing another storage vendor; confirm current plan pricing/limits before creating credentials or storing production objects.
- Implement an S3-compatible adapter for get, put, paginated list, and prefix deletion; preserve JSON content type and object-key semantics.
- Keep storage credentials server-only and use least-privilege access.

### 4. Durable background workflows

- Replace Cloudflare Workflow instances with a durable queue/workflow engine.
- Preserve stable run IDs, idempotency, retries, status inspection, cancellation, stuck-run watchdogs, and billing settlement.
- Vercel Cron may trigger schedulers/reconcilers, but must not be treated as a durable workflow engine.
- Select a provider only after its cost, retry semantics, and deployment compatibility are reviewed.

### 5. SAM agent

- Replace Cloudflare Durable Object session runtime with a Node-compatible persistent session service.
- Persist sessions/messages in PostgreSQL and preserve project/org/user authorization before every connection.
- Preserve streaming, tool calls, usage metering, credit holds, compaction, and recovery semantics.
- Verify reconnect and concurrent-session behavior.

### 6. MCP and OAuth

- Keep MCP tool definitions, tenant isolation, API-key auth, and MCP Apps.
- Replace Workers OAuth provider and KV persistence with a portable OAuth authorization server and durable PostgreSQL storage.
- Preserve registration, grant, token, revocation, expiry, and daily cleanup behavior.
- Verify authorization-code/PKCE and refresh-token flows end-to-end.

## Current status

- Vercel project `open-seo` exists.
- Neon project `open-seo`, database `openseo`, is connected through encrypted `POSTGRES_DATABASE_URL` in production, preview, and development.
- Neon Object Storage is enabled on the database's default branch; its S3-compatible endpoint is available for implementation evaluation.
- Existing PostgreSQL migration history is present in the repository.
- No application migration code has been committed yet.
- The selected migration branch currently matches `main`.
- No production deployment has been created and no database migrations have been run.

## Release gates

- [ ] Vercel Node runtime build passes.
- [ ] TypeScript and lint checks pass.
- [ ] Unit and integration tests pass.
- [ ] PostgreSQL migrations reviewed and applied successfully.
- [ ] Authentication and OAuth flows verified.
- [ ] MCP HTTP transport and tenant isolation verified.
- [ ] SAM chat streaming, persistence, and authorization verified.
- [ ] Audit/rank/AI/local workflows verified for retries, idempotency, cancellation, and reconciliation.
- [ ] Object storage operations verified.
- [ ] Cron endpoints authenticated and scheduled.
- [ ] Production environment variables audited without exposing secrets.
- [ ] Preview deployment smoke tests pass before production promotion.
