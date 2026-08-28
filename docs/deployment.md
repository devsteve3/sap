# Deployment Notes — Render

## 1. SPA "Not Found" on view switch (addressed)

**Symptom:** navigating to / reloading a route like `/student/marketplace` returns a
Render 404. React Router routes only resolve after `index.html` loads; a static host
looks for a real file at that path and 404s.

**Fix applied:** `frontend/public/_redirects`
```
/*    /index.html   200
```
Vite copies `public/` into `dist/`, so this ships with the build. Render Static Sites
honor this Netlify-style file with a `200` rewrite (silent, URL unchanged).

**If it still 404s** — confirm the frontend is deployed as a **Static Site** (not a Web
Service), then add a dashboard rule as a fallback:
- Redirects/Rewrites → Source `/*`, Destination `/index.html`, Action **Rewrite** (not Redirect).

This file is ignored if the frontend runs as a **Web Service** (e.g. `vite preview` or a
Node server) — in that case the catch-all must be configured server-side.

## 2. Frontend → backend URL (must fix for production)

`src/service/apiClient.js:6` falls back to `http://localhost:5000` when
`VITE_API_BASE_URL` is unset. On Render, set the build-time env var:
```
VITE_API_BASE_URL=https://<your-backend-service>.onrender.com
```
Without this, the deployed frontend calls localhost and every request fails. See
frontend C2.

## 3. Backend env on Render

- `JWT_SECRET` — set a strong random value (`openssl rand -hex 32`). The code falls back
  to a public hardcoded secret if unset (backend C2/C3).
- `DATABASE_URL` — set a real Postgres connection string, or leave **unset** to use the
  in-memory `pg-mem`. The current `.env` value is a broken template that crashes startup
  (backend H1). Note: pg-mem data does not persist across restarts — use a real Postgres
  for production.
- `PORT` — Render provides this automatically; the server reads `process.env.PORT`.

## 4. Pre-deploy checklist

- [ ] `VITE_API_BASE_URL` points at the live backend (not localhost).
- [ ] Backend `JWT_SECRET` and `DATABASE_URL` set in Render env.
- [ ] `.env` removed from git tracking and added to `.gitignore` (frontend C1).
- [ ] Auth/route guards added before exposing student/firm/university portals (backend C1, frontend H1/H3).
- [ ] `npm run build` locally confirms `_redirects` lands in `dist/`.
