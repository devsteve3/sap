# Backend Issues — `backend/`

Node/Express, ES modules, Express 5. Data layer is a single `query()` abstraction
(`data/db.js`) over real PostgreSQL (`DATABASE_URL`) or seeded `pg-mem`. Passwords
are bcrypt-hashed (rounds 10) and all queries are parameterized — **no SQL injection,
no committed `.env`** (verified). The real problems are auth coverage and config.

---

## Critical

### C1 — Student, firm, and university routes have NO authentication
**`routes/index.js:13-16`**
```js
router.use('/student', studentRoutes);
router.use('/firm', firmRoutes);
router.use('/university', universityRoutes);
```
Only `/admin` is guarded (`routes/index.js:19` — `protect, authorizeRoles('admin')`).
`protect`/`authorizeRoles` are already imported but applied nowhere else. Anyone, with
no token, can:
- `PATCH /firm/applicants/:id` — change any applicant's hiring status (`firmController.js:48`)
- `PATCH /university/logbooks/:id` — approve any logbook (`universityController.js:56`)
- `POST /student/applications` — create applications (`studentController.js:45`)
- Read every tenant's metrics/applicants/logbooks.

**Fix:** mount `protect` (+ appropriate `authorizeRoles`) on the student/firm/university
routers, mirroring the admin line.

### C2 — JWT secret falls back to a hardcoded, public value
**`server.js:12`**
```js
process.env.JWT_SECRET = process.env.JWT_SECRET || process.env.JSON_SECRET_KEY || 'dev-only-insecure-secret';
```
If neither env var is set, tokens are signed/verified with `'dev-only-insecure-secret'`.
In production this lets anyone forge an admin token. No startup check enforces a real secret.

**Fix:** require `JWT_SECRET` at boot; `process.exit(1)` if missing in production. Remove the literal fallback.

### C3 — `.env` JWT secret is the well-known jwt.io sample
**`backend/.env:3`** — the value is the canonical jwt.io HS256 example token (secret
`your-256-bit-secret`). Worthless as a secret.

**Fix:** generate a strong random secret, e.g. `openssl rand -hex 32`, and set it in Render env.

---

## High

### H1 — Broken `DATABASE_URL` template crashes startup
**`backend/.env:4`**
```
DATABASE_URL= ${{ Postgres.DATABASE_URL }}
```
Unsubstituted Railway-style template with a leading space. It is **non-empty**, so
`data/db.js` treats it as truthy, tries to connect with the literal string, fails the
`SELECT 1` check, and aborts via `server.js:46-48` — instead of falling back to pg-mem.

**Fix:** leave `DATABASE_URL` unset locally (uses pg-mem) or set a real connection string. Trim whitespace.

### H2 — Router double-mounted, doubling the API surface
**`server.js:29-30`**
```js
app.use('/api/v1', apiRouter);
app.use('/', apiRouter);
```
Every endpoint is reachable at both `/api/v1/student/...` and `/student/...`. Doubles
attack surface and makes any prefix-scoped auth/rate-limit bypassable via the other path.

**Fix:** mount under a single prefix (`/api/v1`).

### H3 — Error handler leaks internals and infers status wrong
**`middleware/errorMiddleware.js:7-11`**
- `res.statusCode !== 200 ? res.statusCode : 500` can echo a non-error status (201/3xx) set before the throw.
- Returns raw `err.message` (DB/driver errors, SQL text) to clients — information disclosure.

**Fix:** default to 500 unless an explicit error status was set; return a generic message in production, log details server-side.

### H4 — Login does not validate request body
**`controllers/authController.js:40-45`** — no 400 validation; `password || ''`; if
`req.body` is undefined the destructure throws and is caught as a 401 instead of 400.
Inconsistent with `registerUser` (`authController.js:13`) which does validate.

**Fix:** validate `email`/`password` presence, return 400.

---

## Medium

- **M1 — Admin creation under-validated.** `adminController.js:28-43` + `accounts.js:75-82`: admin role only requires `password`; a null username violates `NOT NULL UNIQUE` (`sql/schema.sql:26`) and surfaces as a raw 500. Require `username`; catch unique-violation → 409/400.
- **M2 — Applications hardcoded to the "demo" student.** `studentController.js:58-59` attributes every application to `SELECT id FROM students ORDER BY id LIMIT 1`. After C1, use `req.user.id`.
- **M3 — Student data is global, not per-user.** `studentController.js:8-9,23-30`: metrics/applications span all students. Scope by `req.user.id`.
- **M4 — Firm/university controllers not tenant-scoped.** `firmController.js:21-44`, `universityController.js:22-53` query across all tenants. Add `WHERE firm_id = …` / university scoping after C1.
- **M5 — No enum validation on `status`/`facultySignOff`.** `firmController.js:57-61`, `universityController.js:65-69`: invalid values rely on DB CHECK (raw 500 via H3; pg-mem may not enforce). Validate against an allowed-values list → 400.
- **M6 — CORS wide open.** `server.js:17` — bare `cors()` allows all origins. Configure an allowlist via env once auth tokens are in play.
- **M7 — Non-numeric `:id` returns 404 instead of 400.** `firmController.js:52-54`, `universityController.js:60-62`.

---

## Low

- **L1 — Empty unused `models/` directory.** Dead scaffolding; remove or populate.
- **L2 — Stale comment** in `sql/schema.sql:4` references nonexistent `backend/data/store.js`.
- **L3 — No tests.** `package.json:9` test script is the default failing stub.
- **L4 — `notFound` ordering fragile** because of the double mount (see H2).
- **L5 — `JSON_SECRET_KEY` is an undocumented dead env alias** (`server.js:12`).
- **L6 — 30-day tokens, no revocation** (`authController.js:7`). Password reset does not invalidate existing tokens (DB re-check in `authMiddleware.js:16` does handle deletion).
