# Frontend Issues — `frontend/`

Vite + React 19 + react-router-dom v7. Only the **admin** flow is wired to the real
backend; student/firm/university flows are largely mock/static UI. No secret keys exist
in the client today (good), but several wiring and auth problems block real use.

---

## Critical

### C1 — `.env` is committed and not git-ignored
**`frontend/.gitignore:13`** ignores `*.local` but not `.env`; **`frontend/.env`** is tracked.
Current value (`VITE_API_BASE_URL=http://localhost:5000`) isn't secret, but any
`VITE_`-prefixed var is inlined into the public client bundle — committing `.env` invites
future secrets to leak into git history and shipped JS.

**Fix:** add `.env` to `.gitignore`, commit a `.env.example` template, never put secrets behind `VITE_`.

### C2 — Production build defaults to the local mock server
**`src/service/apiClient.js:6`**
```js
baseURL: import.meta.env.VITE_API_BASE_URL || 'http://localhost:5000'
```
`.env` pins this to `localhost:5000` (the `mock-server.cjs` port). On Render this must be
overridden with the real backend URL, or every API call hits localhost and fails. The `||`
fallback also silently points at localhost if the var is missing at build time.

**Fix:** set `VITE_API_BASE_URL` to the deployed backend URL in Render's build env; consider throwing if unset in production rather than falling back.

### C3 — `firmService` calls paths that 404 (missing `/api/v1` prefix)
**`src/service/firmService.js:7,13,19`** call `/firm/metrics`, `/firm/applicants`,
`/firm/applicants/:id`. The real backend and the mock both expose these under `/api/v1/...`.
`studentService`/`universityService` correctly use `/api/v1/...`; firm does not.

**Fix:** standardize all service paths to `/api/v1/...` and verify each route exists server-side.

---

## High

### H1 — No protected routes; every dashboard is publicly reachable
**`src/route/AppRoute.jsx:38-51`** mounts `/student`, `/firm`, `/university`, `/admin`
with no guard. Only `AdminDashboard` self-checks a token (`AdminDashboard.jsx:60-66`).
The admin "Impersonation Gateways" buttons (`AdminDashboard.jsx:144-162`) navigate to
dashboards that never verify a token.

**Fix:** add a `ProtectedRoute` wrapper (token + role check) around all dashboard routes.

### H2 — Broken import in `useStudentDashboardData` hook
**`src/features/student/hooks/useStudentDashboardData.js:2`** imports from
`'../../../services/student.services'` — wrong folder (`service`, singular) and wrong
filename. Also calls `getDashboardMetrics()`/`getRecentApplications()` which don't exist
(`studentService` exports `getMetrics`/`getApplications`). Harmless only because the hook
is never imported (dead code); an immediate build failure if wired in.

**Fix:** delete the hook, or fix the import path and method names.

### H3 — Fake authentication on student/firm/university login
**`StudentAuth.jsx:22-25`, `FirmAuth.jsx:22-25`, `UniversityAuth.jsx:22-25`** — `handleSubmit`
just `preventDefault()` then `navigate('/...')`. No credentials sent, no token stored, no
validation. 3 of 4 logins are decorative.

**Fix:** wire to real auth endpoints like `adminService.login` (`adminService.js:10-18`), store token, gate routes (H1).

### H4 — JWT stored in `localStorage`
**`apiClient.js:17`** reads, **`adminService.js:15`** writes `localStorage.token`.
Readable by any injected script (XSS) and persists indefinitely.

**Fix:** prefer httpOnly cookies set by the backend; otherwise add expiry handling + strict CSP.

---

## Medium

- **M1 — `_redirects` format vs Render.** `public/_redirects` uses Netlify syntax (`/*  /index.html  200`). Render Static Sites do support this file and the `public/` location is correct (Vite copies it to `dist/`). If Render doesn't pick it up, add a dashboard Rewrite rule (`/*` → `/index.html`, Action: Rewrite). See [deployment.md](./deployment.md).
- **M2 — Leftover mock-server scaffolding shipped in repo.** `mock-server.cjs`, `db.json`, `routes.json`, the `"api"` script (`package.json:8`), and the `json-server` devDep (`package.json:31`). `routes.json` is stale/unused. Confusing dead infrastructure that documents localhost as the backend — move to dev-only or delete.
- **M3 — Large amount of dead/duplicate code.** Duplicate `FirmDashboard` (`features/student/views/FirmDashboard.jsx` is a never-imported stray copy). Never-imported: `MetricCard`, `Modal`, `Input`, `Navbar`, `Sidebar`, `navigationConfig`, `ApplicationRow`, `StudentAuditRow`, and the H2 hook. Several reference a nonexistent `/student/placements` route (`navigationConfig.jsx:15`; real route is `/student/marketplace`) and Tailwind classes/props not defined in `Button.jsx:4-8`. Delete dead files; fix routes/classes for any intended ones.
- **M4 — Marketplace apply sends `undefined`.** `StudentMarketplace.jsx:29-30` uses `placement.company`, but data uses `companyName` (`db.json:95`). Use `placement.companyName` consistently.
- **M5 — Silent failures, no error UI.** `StudentMarketplace.jsx:17,38-40` only `console.error`; fetch failure shows an empty grid, apply failure shows nothing (success uses blocking `alert()` at line 37). Dashboards use hardcoded arrays so have no loading/error states. Add error UI; replace `alert()` with a toast/inline message.
- **M6 — `Input.jsx` id-generation bug.** `Input.jsx:12` wraps the expression in a template literal, so every input without an explicit `id` gets the literal string id, breaking `htmlFor`. Fix: `const inputId = id || label?.toLowerCase().replace(/\s+/g, '-');` (currently unused but real).

---

## Low

- **L1 — Pointless `useMemo`.** `StudentDashboard.jsx:8-28` memoizes over arrays re-allocated every render; recomputes anyway.
- **L2 — Inconsistent logout.** `DashboardLayout.jsx:99-108` vs unused `Sidebar.jsx:11-14` clear different keys and route differently.
- **L3 — Placeholder metadata.** `index.html:7` `<title>frontend</title>`, references missing `/favicon.svg`; README is the untouched Vite template.
- **L4 — Debug `console.error` in shipping code** (`StudentMarketplace.jsx:17,39`).
- **L5 — `<a href>` for internal nav** causes full reloads, e.g. `StudentAuth.jsx:169-171`, `AdminAuth.jsx:128`, `StudentDashboard.jsx:184`, `AppRoute.jsx:58`. Use `<Link>`/`<NavLink>`.
