# Chubb APAC Claims Management - Frontend

Angular 22 single-page application for the Chubb APAC claims platform. One codebase serves four kinds of
people - **claimants, claims officers, managers and administrators** - with a role-aware menu, role-guarded routes and a
screen set per role. It consumes the three existing Spring Boot microservices; **the backend is not modified.**

| | |
|---|---|
| Framework | Angular 22 (standalone components, signals, zoneless change detection, lazy-loaded feature areas) |
| UI | Angular Material 22 (Material 3 theme), responsive for desktop, tablet and mobile |
| Charts | Chart.js 4 through `ng2-charts` 11 |
| Auth | JWT from `POST /api/v1/auth/login`, kept in `sessionStorage` |
| Tests | Vitest through `ng test` |

Documents: [`endpoints_details.md`](endpoints_details.md) (the REST contract this app uses) ·
[`docs/PROJECT_STRUCTURE.md`](docs/PROJECT_STRUCTURE.md) (how the code is organised) · [`CLAUDE.md`](CLAUDE.md) (development notes).

## What each role gets

| Role | Screens |
|---|---|
| **Claimant** | Dashboard (counts per status, recent claims) · Report a claim · My claims (filter, page) · Claim details (progress tracker, **Status / Assignment / Update history tabs**, information requests) · **Claim edit page** (only while `SUBMITTED`) · Information requests (read the question, answer it) · Profile |
| **Claims officer** | Dashboard (queue and workload) · Claims queue (take a claim) · Assigned claims · **Review workspace**: start/resume review with a note; while `UNDER_REVIEW`, **Approve, Reject and Request Information side by side** (Approve/Reject inline, Request Additional Information as a dialog); the **Information Requests table**; history tabs · Settlement page · Assessment history |
| **Manager** | Overview (outstanding claims, claim count, liability per currency, charts) · **Claims** (every claim) · Assignment dashboard (assign / reassign, officer workload) · Officer metrics (workload, approvals, settlements, rejections; charts, sortable table) · Liability reports (per-currency, CSV export) · read-only claim overview with history tabs |
| **Admin** | Dashboard (users by role, active/inactive) · User management (paged list, activate/deactivate) · Create user · User detail (activate / deactivate) · **Edit user** (change role) |
| **Everyone** | Sign in, self-service claimant registration, profile, header notification area, toast messages, confirmation dialogs, empty/error/loading states |

Route map: `/login` · `/register` · `/claimant/**` · `/officer/**` · `/manager/**` · `/admin/**` · `/profile`. A signed-out visit to
any protected URL goes to `/login?returnUrl=...` and returns there after sign-in. A signed-in person who opens another role's
URL is redirected to their own home with a notice. Details are in [`docs/PROJECT_STRUCTURE.md`](docs/PROJECT_STRUCTURE.md#routing-and-security).
Which endpoint feeds which screen (and the route of each) is tabulated in [`endpoints_details.md`](endpoints_details.md#endpoint-to-screen-mapping).

## Prerequisites

- **Node.js** 22.12+ (developed on Node 26) and **npm** 10+ (developed on npm 11).
- The three backend services running locally (below). Java 17 + Maven are only needed to start the backend.

## Setup

```powershell
npm install
```

> **OneDrive:** this folder is inside a OneDrive-synced path. `node_modules` is tens of thousands of files; pause sync, or move the
> project outside OneDrive, if installs or builds feel slow. npm 11 may print an `allow-scripts` warning for `esbuild`, `lmdb` and similar packages; the build works
> without running their install scripts.

## Run

### 1. Start the backend (once, in its own terminal)

The three services must share one `JWT_SECRET` (at least 32 characters) and start in order: User (8185), Claim (8181), Assessment (8182). From the backend repository
(`chubb_Assessment`):

```powershell
.\microservices\scripts\start-all.ps1          # builds (tests skipped) and starts all three; generates a random JWT secret for the run
.\microservices\scripts\stop-all.ps1           # stops what it started
```

Health check: `http://localhost:8185/actuator/health` (and 8181, 8182) must answer `{"status":"UP"}`. Swagger UI: `http://localhost:<port>/swagger-ui/index.html`.

### 2. Start the frontend

```powershell
npm start            # ng serve, http://localhost:4200 (uses proxy.conf.json)
```

Open <http://localhost:4200>. In development the login page shows buttons that fill in the backend's seeded demo accounts:

| Username | Role | Password |
|---|---|---|
| `claimant1` | Claimant | `password123` |
| `officer1` | Claims officer | `password123` |
| `manager1` | Manager | `password123` |
| `admin1` | Admin | `password123` |

These exist only while the backend runs with `DEMO_USERS=true` (its default). They are a demonstration convenience, not real security; the buttons are
hidden in a production build.

### Other commands

| Command | Does |
|---|---|
| `npm test` | unit tests (Vitest, watch mode) · `npm run test:ci` for a single run (no backend needed) |
| `npm run build` | production build into `dist/chubb-claims-ui` (applies `environment.prod.ts` and the bundle budgets) |
| `npx ng build --configuration development` | unminified build, for debugging |
| `npm run e2e:contract` | ~90 checks of every API the app uses against the **live backend**, through the proxy (needs backend + `npm start`) |
| `npm run e2e:ui` | ~50-step browser test of the real screens across all four roles (needs backend + `npm start` + Chrome). See [`e2e/README.md`](e2e/README.md) |

## Integrating with the backend microservices

| Service | Port | Browser prefix | Proxied to |
|---|---|---|---|
| User (Auth) | 8185 | `/svc/user` | `http://localhost:8185` |
| Claim | 8181 | `/svc/claim` | `http://localhost:8181` |
| Assessment | 8182 | `/svc/assessment` | `http://localhost:8182` |

- **Why a proxy:** the backend sends **no CORS headers** and has no gateway, so a page on `localhost:4200` calling `localhost:8181` directly would be blocked by the browser.
  `proxy.conf.json` makes the dev server forward each prefix to its service (stripping the prefix), so the browser only ever talks to its own origin. No backend change is needed.
- **Where each call goes:** all three services are reached through `src/app/config/api.config.ts` (`API.user`, `API.claim`, `API.assessment`), used by the typed services in
  `src/app/services/`. The full endpoint list, with the service method that calls each, is in [`endpoints_details.md`](endpoints_details.md).
- **Authentication:** `auth.service.ts` posts to the User Service, stores `{ accessToken, username, role, expiresAt }` in `sessionStorage` (not `localStorage`: it dies with the tab), and
  `jwt.interceptor.ts` adds `Authorization: Bearer <token>` to every call to the three prefixes (never to login/register, never to any other URL). The same token is accepted by all three services. When the token expires - or any call answers 401 - the person is signed out
  and sent to login.
- **Different ports or hosts:** change the `target` values in `proxy.conf.json` (development) or the web server's routing (production). If you would rather call the services
  directly (for example after adding CORS to the backend), set the three URLs in `src/environments/environment*.ts`.
- **Errors:** the backend's JSON error body (`{ timestamp, status, error, code, message, path }`) is turned into one toast by `error.interceptor.ts`; forms also check the same rules client-side first.

### Production hosting

`ng build` produces static files. The web server must serve `index.html` for unknown paths (single-page app) **and** forward the same three prefixes - the build keeps them identical to
development, so no rebuild is needed per environment. Example for nginx:

```nginx
server {
  listen 80;
  root /usr/share/nginx/html;                       # contents of dist/chubb-claims-ui/browser
  location / { try_files $uri $uri/ /index.html; }
  location /svc/user/       { proxy_pass http://user-service:8185/; }        # trailing slash strips the prefix
  location /svc/claim/      { proxy_pass http://claim-service:8181/; }
  location /svc/assessment/ { proxy_pass http://assessment-service:8182/; }
}
```

Serve it over HTTPS: the backend itself uses plain HTTP and a shared signing secret, and its own README says it is not production-ready.

## Behaviour worth knowing (it comes from the backend)

- **Approve, then settle.** Approval (with an amount) and settlement are two steps; `APPROVED` is not final. A rejected claim can never be settled.
- **The officer is the token.** An officer's actions are recorded against their own login; the UI never sends an officer name for review, request, decision or settlement.
- **Visibility.** A claimant sees only their own claims; an officer sees the unassigned queue plus their own. A claim outside that scope answers *404*, exactly like one that does not exist.
- **Information requests are text only.** There is no document upload in the backend, so there is none in the UI.
- **Liability is per currency.** Amounts in different currencies are never added; the manager screens show one figure per currency and no cross-currency total or chart.
- **A manager cannot list officers.** `GET /users` is admin-only, so the assign dialog suggests the officers found in the workload report and accepts any typed login name; the server checks that it is an active claims officer.
- **Deactivating or re-roling someone does not end their open session.** The token keeps working until it expires (the backend has no revocation). A role change shows at the person's next sign-in.

## Known limitations

- **Notifications** are client-side only (toasts plus the header bell, which also shows "needs your attention" counts at sign-in). Server-pushed notifications are the planned Kafka enhancement
  (see `CLAUDE.md`); `NotificationService.push()` is the single place to feed them.
- **Light theme only** (no dark mode). Charts use the light palette.
- **No server-side sorting** on claim lists (the backend returns newest first and offers no sort parameter); the officer-metrics table sorts in the browser.
- **Admin dashboard counts** read `GET /users` page by page (100 at a time, up to 2,000 users) because the backend has no count-by-role.
- **Not production-ready as a system:** tokens live in `sessionStorage` (readable by script), the backend uses a shared signing secret and has no refresh or revocation, and the demo accounts are on by default.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Login shows "Cannot reach the server" | A backend service is not running, or the proxy target port is wrong. Check the three `/actuator/health` URLs. |
| Every call is `401` right after login | The services were started with **different** `JWT_SECRET` values. Restart all three with the same one (`start-all.ps1` does this). |
| `404` on `/svc/...` in the browser console | You are not running through `npm start` (the proxy only exists in the dev server), or the production web server lacks the three `location` blocks. |
| Port 8181 already in use | The backend's original monolith also defaults to 8181. Stop it, or start the Claim Service on another port and update `proxy.conf.json`. |
| Icons show as words | The icon font did not load; run `npm install` so `material-icons` is present (fonts are bundled, no CDN needed). |
