# Student Connect — Issues & Remediation Docs

This folder documents the main issues found during a full inspection of the
`backend/` (Node/Express + PostgreSQL/pg-mem) and `frontend/` (Vite + React 19 +
react-router-dom v7) on **2026-06-17**.

## Documents

| Doc | Scope |
|-----|-------|
| [backend-issues.md](./backend-issues.md) | Express API: auth coverage, JWT secret, DB config, error handling |
| [frontend-issues.md](./frontend-issues.md) | React app: API wiring, fake auth, dead code, bugs |
| [deployment.md](./deployment.md) | Render hosting: SPA routing fix, env vars, backend URL |

## Severity legend

- **Critical** — security hole or breaks the app in production. Fix before next deploy.
- **High** — broken functionality or significant risk.
- **Medium** — bugs, confusion, tech debt that will bite soon.
- **Low** — cosmetic, cleanup, polish.

## Fix in this order

1. **Backend C1** — student/firm/university routes have **no authentication**. Anyone can mutate any tenant's data.
2. **Backend C2/C3** — JWT secret falls back to a public hardcoded string; `.env` secret is the well-known jwt.io sample.
3. **Backend H1** — `DATABASE_URL` in `.env` is an unsubstituted template; it is truthy and crashes startup instead of falling back to pg-mem.
4. **Frontend C2** — production build defaults the API base URL to `localhost:5000` (the mock server). Must be set to the real backend URL on Render.
5. **Frontend C3** — `firmService` calls `/firm/*` without the `/api/v1` prefix; 404s even against the mock.
6. **Frontend C1** — `.env` is committed and not git-ignored.
7. **Frontend H1/H3** — no route guards; 3 of 4 logins are decorative (just `navigate()`).
8. Cleanup: dead code, duplicate components, leftover mock-server scaffolding.

> Note: the Render "Not Found on view switch" issue you hit has already been
> addressed via `frontend/public/_redirects`. See [deployment.md](./deployment.md)
> to confirm it's picked up.
