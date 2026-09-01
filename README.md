# SNAC Close Acceleration Estimate (`snac-cap-est`)

Internal Block tool for reviewing accounting estimate journal entries during month-end close. Built on the golden path stack: React + TypeScript frontend, Node.js BFF, Kotlin/Misk service.

> **Internal Block repository** — `squareup/snac-cap-est` is private. Code here must stay in a Block-managed GitHub org.

## Status

This is an early-stage prototype. **Most of the UI runs on mock data**, not live services:

- The BFF serves hardcoded responses for `/api/v1/me`, `/api/v1/journal_entries`, and `/api/v1/check_issuance/uploaded_files`.
- The frontend carries its own fixtures in `frontend/src/mockData.ts`.
- The Kotlin service is a skeleton — one controller, one model, no persistence wired up.

Treat screenshots and demos as illustrative until the service layer lands.

## Stack

| Layer | Tech | Port |
|---|---|---|
| Frontend | React 18, TypeScript, Vite 5, Base Web (`baseui`), Styletron, SWR, React Router 6, Sentry | 5173 |
| BFF | Node.js, Express 4, `http-proxy-middleware` | 3001 |
| Service | Kotlin 1.9, Misk `2023.12.01.0`, Gradle, MySQL connector, Wire | 8080 |

Frontend and BFF are Yarn workspaces (`@snac/frontend`, `@snac/bff`). The Kotlin service is a separate Gradle build under `service/`.

## Prerequisites

- Node.js 18+ and Yarn
- JDK 17 (the Gradle build sets `jvmToolchain(17)`) — only needed if you run the Kotlin service

## Running locally

Install once from the repo root, then start the frontend and BFF together:

```bash
yarn install
yarn dev
```

`yarn dev` runs both workspaces concurrently. Open **http://localhost:3001** — not the Vite port. In development the BFF proxies every non-API request through to Vite (including websockets for HMR), so port 3001 is the single entry point.

The Kotlin service is optional and runs separately:

```bash
yarn service   # → cd service && ./gradlew run
```

Without it, the BFF's mock endpoints cover the routes the UI currently reads. Anything else under `/api/v1` proxies to `SERVICE_URL` and will fail until the service is up.

### Environment

The BFF reads these (all optional, defaults shown):

| Variable | Default | Purpose |
|---|---|---|
| `PORT` | `3001` | BFF listen port |
| `SERVICE_URL` | `http://localhost:8080` | Kotlin service to proxy `/api/v1` to |
| `VITE_URL` | `http://localhost:5173` | Dev-only Vite target |
| `NODE_ENV` | — | Set to `production` to serve `frontend/dist` instead of proxying to Vite |

`dotenv` is loaded, so a `bff/.env` file works.

## Routes

Three pages are currently wired up in `frontend/src/components/Router/Router.tsx`. `/` redirects to `/cap-est`.

| Path | Page | What it shows |
|---|---|---|
| `/cap-est` | `JournalEntriesPage` | Estimate journal entries — period, amount, GL segment/account, estimate vs. actual, approval/posting/reversal state, variance |
| `/cap-est/lookup` | `LookupTablesPage` | Lookup keys and values with expandable rate history |
| `/cap-est/prd` | `PrdReferencePage` | PRD reference: JE process flow diagram, estimation and cutoff GURs, reversal policy |

### Unwired leftovers

`frontend/src/config/routes.ts` defines a much larger surface than the app serves — remittances, claims, search, check-issuance forms — and `frontend/src/pages/` holds **13 page directories where only 3 are routed**. These are inherited from the golden path template and are not reachable. Expect to delete or adopt them rather than assume they work.

## API surface

Served by the BFF (`bff/src/index.ts`):

```
GET  /api/v1/me                                mock user
GET  /api/v1/journal_entries                   mock journal entries
GET  /api/v1/check_issuance/uploaded_files     mock, empty page
GET  /bff/automation-jobs                      aggregates from Kotlin service
GET  /_status                                  health check
*    /api/v1/*                                 proxied → SERVICE_URL
```

`/bff/automation-jobs` returns `502 Upstream service unavailable` when the Kotlin service is down.

## Layout

```
.
├── frontend/          @snac/frontend — Vite + React
│   └── src/
│       ├── components/    Router, Sidebar, UserAvatar
│       ├── config/        routes.ts — APP_PATHS / API_PATHS
│       ├── pages/         13 dirs, 3 routed
│       └── mockData.ts    JournalEntry, LookupEntry, AuditEntry fixtures
├── bff/               @snac/bff — Express proxy + mock API
│   └── src/
│       ├── index.ts       server, mocks, proxy config
│       └── routes/        automationJobs.ts
├── service/           Kotlin/Misk — com.squareup.snaccapest
│   └── src/main/kotlin/com/squareup/snaccapest/
│       ├── Main.kt
│       ├── controllers/AutomationJobsController.kt
│       ├── models/UploadedFile.kt
│       └── modules/SnacCapEstModule.kt
└── package.json       Yarn workspace root
```

## Build and test

```bash
yarn build                      # frontend then bff
yarn workspace @snac/frontend test    # Jest
cd service && ./gradlew test          # Kotlin
```

## Known gaps

- **No `.gitignore` in the repo.** `node_modules/` is currently ignored only by the user-level `~/.gitignore` on this machine. On a fresh clone elsewhere — a teammate's laptop or CI — it will show as untracked and can be committed by accident. Adding a repo-level `.gitignore` covering `node_modules/`, `dist/`, `build/`, and `.env` should be an early fix.
- Mock data is duplicated between `bff/src/index.ts` and `frontend/src/mockData.ts`, and the two shapes have already drifted (the BFF sends `glSegment` / `glAccount`; the frontend interface declares a single `glString`).
- The Kotlin service has no tests and no database schema.
