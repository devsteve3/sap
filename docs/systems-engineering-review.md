# Systems Engineering Review — Student Connect

A higher-altitude companion to [backend-issues.md](./backend-issues.md),
[frontend-issues.md](./frontend-issues.md), and [deployment.md](./deployment.md).
Those docs catalogue code-level defects; this one looks at the system as an
operated service — reliability, data durability, configuration, observability,
deployment, scalability, and maintainability — on **2026-06-19**.

**Stack:** Express 5 (ES modules) + `pg`/`pg-mem` backend, Vite + React 19 SPA
frontend, hosted on Render. No containers, no CI, no tests, no monitoring.

---

## Severity legend
- **S1 — Critical:** data loss, outage, or security breach in normal operation.
- **S2 — High:** unreliable under load/failure, or blocks a safe production deploy.
- **S3 — Medium:** operability/maintainability gap that will cause incidents over time.
- **S4 — Low:** hygiene and polish.

---

## 1. Data & Persistence

### S1 — Default datastore is ephemeral; all data is lost on restart
`backend/data/db.js:29-38`: when `DATABASE_URL` is unset the app runs `pg-mem`,
an **in-process, in-memory** Postgres seeded from `sql/`. Every deploy, crash,
or Render dyno cycle wipes all users, applications, and logbooks. There is no
persistence in the default mode and no warning at the API boundary that data is
volatile.
**Fix:** provision a real managed Postgres (Render Postgres / Neon / RDS), set
`DATABASE_URL`, and treat pg-mem strictly as a local-dev/test fixture.

### S2 — No migration runner; schema lifecycle is manual
`sql/schema.sql`, `sql/seed.sql`, and `sql/migrate_admins.sql` are applied
automatically **only** to pg-mem (`db.js:32-33`). Against a real database they
must be run by hand (`db.js:10-11` says as much). There is no migration tool
(Flyway/Prisma/node-pg-migrate), no version table, and no ordering guarantee
between `schema`/`seed`/`migrate_admins`. Schema drift between environments is
inevitable.
**Fix:** adopt a migration tool with a `migrations` table; run it on deploy.

### S3 — No backup / restore / retention strategy
No documented backups, PITR, or restore drill. Once a real DB exists this is
mandatory for a system holding student/firm/university records (and likely PII).

### S3 — Connection pool is unbounded and unconfigured
`db.js:26` creates `new Pool({ connectionString })` with default sizing and **no
`error` handler** on the pool. A dropped backend connection emits an
`'error'` event with no listener, which can crash the process; default pool size
may not match the DB's connection limit.
**Fix:** set `max`, idle/timeout options, and attach `pool.on('error', …)`.

---

## 2. Reliability & Availability

### S2 — No graceful shutdown
`server.js:42-44` calls `app.listen` and never handles `SIGTERM`/`SIGINT`.
Render sends `SIGTERM` on every deploy and scale event; without a handler,
in-flight requests are killed and the pg pool is not drained. This causes
dropped requests and connection leaks on each deploy.
**Fix:** on `SIGTERM`, stop accepting new connections (`server.close`), drain,
then `pool.end()`.

### S2 — Startup fails late and opaquely on bad config
`db.js:24` treats any non-empty `DATABASE_URL` as valid, so the broken template
value in `.env` (`${{ Postgres.DATABASE_URL }}`, see backend H1) passes the
truthiness check and only fails at `pool.query('SELECT 1')`, aborting via
`server.js:46-48`. There is no config validation step and no distinction between
"misconfigured" and "DB down."
**Fix:** validate required env at boot (see §4) and fail with a clear message.

### S3 — No process supervisor / restart policy documented
Single `node server.js` process (`package.json` start script). No clustering, no
PM2/systemd, reliance entirely on Render's restart behavior. An unhandled
rejection or the pool-error above takes the whole service down with no in-process
recovery.
**Fix:** add `process.on('unhandledRejection'/'uncaughtException')` logging +
controlled exit; document the platform restart policy.

### S3 — Health check does not reflect readiness
`server.js:27` `GET /` returns a static string regardless of DB state. A load
balancer hitting it will route traffic to an instance whose database is
unreachable.
**Fix:** add `/healthz` (liveness) and `/readyz` (runs `SELECT 1`) endpoints.

---

## 3. Security & Secrets *(operational lens — see backend/frontend docs for code detail)*

### S1 — JWT signing secret has an insecure fallback
`server.js:12` falls back to a hardcoded `'dev-only-insecure-secret'`, and the
committed `.env` uses the public jwt.io sample (backend C2/C3). Forged admin
tokens are trivial if the env var is ever missing.
**Fix:** require a strong `JWT_SECRET` at boot; never embed a literal.

### S2 — No edge hardening: no security headers, no rate limiting, open CORS
- No `helmet` (or equivalent) — missing HSTS, X-Content-Type-Options, etc.
- No rate limiting on `/auth/login` → unthrottled credential stuffing/brute force.
- `cors()` allows all origins (`server.js:17`).
- No request body size cap beyond Express defaults.
**Fix:** add `helmet`, an auth rate limiter, a CORS allowlist via env, and explicit
`express.json({ limit })`.

### S2 — Tokens in `localStorage`, long-lived, non-revocable
Frontend stores JWTs in `localStorage` (XSS-exfiltratable), tokens live 30 days
(`authController.js:7`) with no revocation/refresh. Operationally there is no way
to force-logout a compromised account short of deleting it.
**Fix:** httpOnly cookies + short access tokens + refresh, or a revocation list.

### S3 — Secret management is ad hoc
`.env` committed on the frontend (frontend C1); no `.env.example`, no documented
secret inventory, no rotation procedure.
**Fix:** untrack `.env`, add `.env.example`, document required secrets per service.

---

## 4. Configuration Management

### S2 — No environment validation or schema
Env vars (`JWT_SECRET`, `DATABASE_URL`, `PORT`, frontend `VITE_API_BASE_URL`) are
read ad hoc with silent fallbacks (`server.js:12,36`, `apiClient.js:6`). Missing
or malformed config surfaces as runtime failures, not startup errors.
**Fix:** validate config at boot (e.g. `zod`/`envalid`); fail fast with a list of
what's missing.

### S2 — Frontend defaults its API at build time to `localhost`
`apiClient.js:6` falls back to `http://localhost:5000`. Because Vite inlines env
at **build** time, a missing `VITE_API_BASE_URL` ships a broken bundle that calls
localhost (frontend C2). Config errors are baked into the artifact, not catchable
at runtime.
**Fix:** set the var in Render's build env; throw at build/startup if absent in prod.

### S3 — Duplicate, unversioned route mount
`server.js:29-30` mounts the router at both `/api/v1` and `/`. Two public
contracts to maintain; any gateway policy (auth, rate limit, WAF rule) keyed to
one prefix is bypassable via the other.
**Fix:** single canonical prefix.

### S4 — Node runtime not pinned
No `engines` field or `.nvmrc` in either `package.json`. Build/runtime Node
version is whatever the platform defaults to → reproducibility risk.

---

## 5. Observability

### S2 — Logging is unstructured `console.log`, no levels, no correlation
`server.js:21-23` logs `method + url` via `console.log`; errors go to
`console.error` (`server.js:47`, `errorMiddleware`). No log levels, no JSON,
no request IDs, no timestamps from the app. Impossible to trace a request across
the stack or filter by severity in aggregation.
**Fix:** structured logger (pino/winston) with levels and a per-request id.

### S2 — Error handler leaks internals to clients
`middleware/errorMiddleware.js:7-11` returns raw `err.message` (DB/driver text)
and can echo a non-error status set before the throw (backend H3). This is both
an info-disclosure and an observability anti-pattern (errors masked as 2xx/3xx).
**Fix:** generic client message + full detail to logs; default unknown errors to 500.

### S3 — No metrics, tracing, or alerting
No `/metrics`, no APM, no uptime/error-rate alerting. Incidents are discovered by
users, not signals.
**Fix:** export basic RED metrics and wire an uptime/error alert.

---

## 6. Deployment & CI/CD

### S2 — No CI pipeline
No `.github/` (or other CI). Nothing runs lint, build, or tests before deploy;
no protection against shipping a broken `main`.
**Fix:** add a pipeline: install → lint → build → test on PR; gate merges.

### S2 — No automated tests anywhere
Both `package.json` `test` scripts are the failing default stub
(backend `package.json`, frontend `package.json:9`). Zero unit/integration/e2e
coverage on an app handling auth and multi-tenant data.
**Fix:** start with auth + route-guard integration tests (high value given the C1
class of bug just fixed).

### S3 — No containerization / reproducible build
No `Dockerfile`/`.dockerignore`. Deploys depend on Render's buildpack inference;
local/prod parity is not guaranteed.
**Fix:** add Dockerfiles (or a committed `render.yaml`) to pin the build.

### S3 — Deploy artifacts mixed with app; mock server shipped in frontend
`mock-server.cjs`, `db.json`, `routes.json`, and `json-server` live in the
deployed frontend package (frontend M2). Dev-only tooling in the production
artifact increases bundle/install surface and documents localhost as the backend.
**Fix:** isolate dev tooling; remove from the production dependency/build path.

### S4 — No `render.yaml` (infra as code)
Render config (static-site rewrite, env vars, services) lives only in the
dashboard. The SPA fix currently relies on `public/_redirects` (deployment.md).
**Fix:** commit a `render.yaml` describing both services and the rewrite rule.

---

## 7. Scalability & Performance

### S3 — Every authenticated request hits the database to rehydrate the user
`authMiddleware.js:16` calls `findAccountById` on **every** protected request,
adding a DB round-trip per call with no caching. Under load this multiplies DB
connections (compounding the unbounded pool, §1).
**Fix:** cache by token/user for a short TTL, or trust verified JWT claims for the
hot path and only re-check on sensitive operations.

### S3 — Per-tenant queries are unscoped and unindexed-by-design
Controllers query across all rows (`ORDER BY id LIMIT 1`, full-table counts —
backend M2/M3/M4). Beyond the correctness bug, these full scans don't scale and
won't benefit from tenant-scoped indexes.
**Fix:** scope by `req.user.id` (now possible post-C1) and index the tenant keys.

### S4 — No pagination on list endpoints
`/firm/applicants`, `/university/logbooks/pending`, `listAllAccounts` return
unbounded result sets. Fine at seed scale, a latency/memory problem as data grows.
**Fix:** add limit/offset or cursor pagination.

---

## 8. Maintainability

### S3 — Significant dead/duplicate code and empty scaffolding
Empty `backend/models/` dir (backend L1); ~8 never-imported frontend components,
a duplicate `FirmDashboard`, and stale `routes.json` (frontend M3/M2). Raises
cognitive load and hides what is actually wired.
**Fix:** delete dead code; keep the tree representative of running behavior.

### S4 — Stale/misleading documentation in source
`sql/schema.sql:4` references a nonexistent `backend/data/store.js`; both READMEs
are largely template/placeholder (frontend L3).
**Fix:** correct comments; write real service READMEs (run, env, deploy).

---

## Recommended remediation order (systems lens)

1. **Persist data** — provision real Postgres + migration runner (§1 S1/S2). Nothing else matters if data evaporates on restart.
2. **Lock down secrets & config** — enforce `JWT_SECRET`, fix `DATABASE_URL`, validate env at boot, untrack `.env` (§3, §4).
3. **Make deploys safe** — graceful shutdown, `/readyz`, pool error handling (§1, §2).
4. **Get a safety net** — CI running lint/build/auth tests before merge (§6).
5. **Harden the edge** — helmet, rate limiting, CORS allowlist, single API prefix (§3, §4).
6. **See what's happening** — structured logs + a basic uptime/error alert (§5).
7. **Then** scalability and cleanup (§7, §8).
