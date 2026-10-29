# Navora — City Explore & Travel Planning Platform

Navora helps people discover places in a city and turn them into a realistic plan. A user
can browse and search places near them, read and write reviews, and ask Navora to plan a
trip for a number of days. Navora picks places that match the user's interests, budget and
practical needs (pet-friendly, parking, wheelchair access), puts them into days, checks
opening hours, visit time and travel time, orders the stops to reduce travel, and shows the
route on a map. Local users can submit new places, which moderators approve before they go
public.

The default city is **Dhaka** (timezone `Asia/Dhaka`, currency `BDT`), and the demo data
contains about 44 well-known Dhaka places.

| | |
|---|---|
| **Live app** | `<add the Vercel link>` |
| **Live API** | `<add the Render link>` (health check: `/ready`) |
| **API contract** | [`backend/docs/openapi.yaml`](backend/docs/openapi.yaml) (OpenAPI 3.1) |
| **Docs** | [System design](docs/SYSTEM_DESIGN.md) · [Project report](docs/PROJECT_REPORT.md) · [Contributing](CONTRIBUTING.md) |

---

## Features

**Discover**
- Places near you or near any point on the map, newest places, and full-text search that tolerates typos
- Filters: category, maximum entry fee, minimum rating
- Place pages with facilities, opening hours, entry fee and community reviews
- Unknown information is shown as *Unknown*, never as "free", "open" or "yes"

**Plan**
- **Flow A, pick then plan:** choose places while browsing, then build a trip around them
- **Flow B, plan by days:** enter days, interests, budget, start point and needs; Navora chooses the places
- Recommendations with a 0–100 score, the reasons behind it, and warnings about missing data
- Day-by-day itinerary: timed stops, travel legs, totals, route on the map
- Edit stops, reorder, remove or add places, re-optimise a day, or rebuild the whole plan; every version is kept
- Mandatory requirements are never quietly relaxed. If too few places qualify, Navora says which requirement is limiting

**Community**
- Submit a place (it goes to *Pending*), edit and resubmit if rejected, and see the moderator's notes
- Rate and review places (one review per person per place); ratings update instantly
- Moderation queue for moderators: approve or reject with a reason

**Optional:** an AI Trip Assistant in the backend that explains a saved itinerary using a
local, free Ollama model. It is off by default and not yet shown in the web app.

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React 19, TypeScript (strict), Vite 7, React Router 7, TanStack Query 5, Leaflet + react-leaflet with OpenStreetMap tiles |
| Backend | Node.js 22.12+ (ESM), Express 5, zod 4 validation, pino logging, helmet, CORS |
| Database | PostgreSQL 15+ with **PostGIS**, `pg_trgm` and `citext`; Knex migrations |
| Cache and rate limits | Upstash Redis (REST) + `@upstash/ratelimit` |
| Auth | JWT access tokens (HS256, 15 min) + rotating single-use refresh tokens; argon2id passwords |
| Routing | Built-in distance estimates (default) or OpenRouteService for real road times |
| AI (optional) | Ollama with an open-weight model (e.g. `llama3.2:3b`) |
| Tests | Vitest, supertest, OpenAPI contract tests (ajv, swagger-parser) |
| CI | GitHub Actions: install, `npm audit`, full test suite against a PostGIS service |
| Hosting | Vercel (frontend), Render (backend), Neon (PostgreSQL + PostGIS), Upstash (Redis) |

## Repository layout

```text
Navora/
├── backend/                Express REST API
│   ├── src/                config, middleware, modules (one per feature), integrations, shared
│   ├── migrations/         database schema (Knex, raw SQL for PostGIS, enums, triggers)
│   ├── seeds/              categories and interest tags
│   ├── scripts/            demo data loader (Dhaka places)
│   ├── tests/              unit, api, integration (PostGIS), contract (OpenAPI)
│   ├── docs/openapi.yaml   API contract
│   ├── docker-compose.yml  local PostgreSQL 17 + PostGIS on port 5433
│   └── .env.example        every backend setting, without secrets
├── frontend/               React web app
│   ├── src/                pages, components, api client, auth, styles
│   ├── .env.example        optional frontend settings
│   ├── SETUP.md            detailed local setup with a first-run checklist
│   └── DEPLOY.md           step-by-step free deployment guide
├── docs/                   system design and project report
├── .github/workflows/      backend CI
├── CONTRIBUTING.md         team workflow and coding conventions
└── README.md               this file
```

---

## Prerequisites

| Tool | Version | Check |
|---|---|---|
| Node.js | 22.12 or newer (LTS) | `node --version` |
| Git | any recent | `git --version` |
| Docker Desktop | any recent, running | `docker --version` |
| Upstash account | free plan | can sign in at [console.upstash.com](https://console.upstash.com) |
| OpenRouteService key | optional, free | only for real road travel times |

Without Docker you can install PostgreSQL with the PostGIS extension yourself (Windows:
*Application Stack Builder → Spatial Extensions → PostGIS*) and point `DATABASE_URL` at it.

## Run it locally

The frontend runs on `http://localhost:5173` and forwards every `/api` request to the backend
on `http://localhost:3000`, so there are no CORS problems in development.

### 1. Clone

```bash
git clone https://github.com/MinazurRahman/Navora.git
cd Navora
```

### 2. Start the database

```bash
cd backend
docker compose up -d        # PostgreSQL 17 + PostGIS on localhost:5433
docker compose ps           # wait until the status is "healthy"
```

### 3. Create a free Upstash Redis database

In [console.upstash.com](https://console.upstash.com): **Create database → Redis**, free
plan, a nearby region. Open it and copy **UPSTASH_REDIS_REST_URL** and
**UPSTASH_REDIS_REST_TOKEN** from the *REST API* section. Each developer uses their own.

### 4. Configure the backend

```bash
npm ci
cp .env.example .env        # Windows PowerShell: Copy-Item .env.example .env
node -e "console.log(require('crypto').randomBytes(48).toString('base64url'))"
```

Open `backend/.env` and set:

| Variable | Value |
|---|---|
| `JWT_SECRET` | the random value printed above (at least 32 characters) |
| `UPSTASH_REDIS_REST_URL` | from step 3 |
| `UPSTASH_REDIS_REST_TOKEN` | from step 3 |
| `DATABASE_URL` | keep the default `postgres://navora:navora_dev_password@localhost:5433/navora` |
| `MAP_PROVIDER` | `estimate` (no key needed), or `openrouteservice` with `MAP_API_KEY` |

The backend validates every setting at startup and refuses to start, naming the problem, if
something required is missing.

### 5. Create the tables and load data

```bash
npm run migrate:latest      # tables, indexes, triggers, PostGIS extensions
npm run seed                # 16 categories and 14 interest tags
npm run seed:demo           # ~44 demo Dhaka places (refuses to run in production)
```

### 6. Start the backend

```bash
npm run dev                 # http://localhost:3000
```

Open `http://localhost:3000/ready`; it should say `"status":"ready"`.

### 7. Start the frontend (second terminal)

```bash
cd frontend
npm ci
npm run dev                 # http://localhost:5173
```

The frontend needs no `.env` locally. To change the backend address or the map tiles, copy
`frontend/.env.example` to `frontend/.env.local`.

### 8. Try it

Open `http://localhost:5173`, create an account, open a place and leave a review, then go to
**Plan a trip**, choose 2 days and an interest, preview the matching places and click
**Build my itinerary**. [`frontend/SETUP.md`](frontend/SETUP.md) has a full first-run
checklist and a troubleshooting table.

### Make a moderator account

There is no API for changing roles (on purpose). Register normally, then:

```bash
cd backend
docker compose exec postgres psql -U navora -d navora \
  -c "UPDATE users SET role = 'moderator' WHERE email = 'you@example.com';"
```

Sign out and back in. A **Moderation** link appears in the top bar.

---

## Environment variables

Full lists with comments are in [`backend/.env.example`](backend/.env.example) and
[`frontend/.env.example`](frontend/.env.example). Real `.env` files are git-ignored and are
shared with the team privately, never through the repository.

**Backend, required**

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | PostgreSQL + PostGIS connection string (`?sslmode=require` for managed databases) |
| `JWT_SECRET` | signs access tokens; at least 32 characters, same on every instance |
| `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN` | Redis for rate limiting and caching |

**Backend, commonly changed**

| Variable | Default | Purpose |
|---|---|---|
| `PORT` | `3000` | HTTP port (do not set on Render) |
| `CORS_ORIGINS` | `http://localhost:5173,http://localhost:3000` | browser origins allowed to call the API |
| `TRUST_PROXY` | `0` | proxy hops in front of the app (`1` on Render) |
| `MAP_PROVIDER` / `MAP_API_KEY` | `estimate` / empty | routing provider and its key |
| `MAP_FALLBACK_TO_ESTIMATE` | `true` | use flagged estimates when the provider fails, instead of 503 |
| `AI_ENABLED`, `OLLAMA_BASE_URL`, `OLLAMA_MODEL` | `false`, `http://localhost:11434`, `llama3.2:3b` | optional AI assistant |

**Frontend**

| Variable | Purpose |
|---|---|
| `VITE_API_BASE_URL` | backend URL for production builds (empty in development) |
| `VITE_DEV_BACKEND_URL` | where the dev server forwards `/api` (default `http://localhost:3000`) |
| `VITE_MAP_TILE_URL`, `VITE_MAP_ATTRIBUTION` | map tile server and its credit line |

## Scripts

| Where | Command | What it does |
|---|---|---|
| `backend/` | `npm run dev` | start the API with auto-restart |
| `backend/` | `npm start` | start the API (production) |
| `backend/` | `npm test` | run all tests |
| `backend/` | `npm run migrate:latest` / `migrate:rollback` / `migrate:status` | database migrations |
| `backend/` | `npm run seed` / `npm run seed:demo` | reference data / demo places |
| `frontend/` | `npm run dev` | dev server with live reload |
| `frontend/` | `npm run build` | type-check and build into `frontend/dist/` |
| `frontend/` | `npm run typecheck` | TypeScript check only |
| `frontend/` | `npm run preview` | serve the production build locally |

## Tests

```bash
cd backend
npm test
```

Unit and API tests need no database. Integration and contract tests run only when
`TEST_DATABASE_URL` points at a separate, disposable PostGIS database (the tests wipe it):

```bash
TEST_DATABASE_URL=postgres://navora:navora_dev_password@localhost:5433/navora_test npm test
```

CI runs the whole suite, including the database tests, on every pull request.

## API at a glance

All endpoints are under `/api/v1` and return `{ success, data, meta }` or
`{ success: false, error: { code, message, details, requestId } }`.

| Area | Endpoints |
|---|---|
| Auth | `POST /auth/register`, `/auth/login`, `/auth/refresh`, `/auth/logout`, `GET /auth/me` |
| Catalogue | `GET /categories`, `GET /tags` |
| Places | `GET /places`, `/places/nearby`, `/places/search`, `/places/:id`, `/places/mine`; `POST`, `PATCH`, `DELETE /places` |
| Moderation | `GET /places/moderation`, `POST /places/:id/approve`, `/places/:id/reject`, `GET /places/:id/moderation-events` |
| Reviews | `GET`, `POST /places/:id/reviews`; `PATCH`, `DELETE /reviews/:id` |
| Recommendations | `POST /recommendations` |
| Trips | `POST`, `GET /trips`; `GET`, `PATCH`, `DELETE /trips/:id` |
| Itineraries | `POST /itineraries/generate`, `GET /itineraries`, `GET`, `PATCH`, `DELETE /itineraries/:id`, `POST /itineraries/:id/regenerate` |
| Routing | `POST /routing/optimize` |
| AI assistant | `POST /ai/trip-assistant`, `GET /ai/trip-assistant/status` |
| Health | `GET /health`, `GET /ready` (outside `/api/v1`) |

The complete contract, with every field, status and error code, is in
[`backend/docs/openapi.yaml`](backend/docs/openapi.yaml). Paste it into
[editor.swagger.io](https://editor.swagger.io) to browse it.

## Deployment

The live version runs on free plans: Vercel (frontend), Render (backend), Neon (PostgreSQL
with PostGIS) and Upstash (Redis), all in the Singapore region.
[`frontend/DEPLOY.md`](frontend/DEPLOY.md) walks through it step by step. On the free Render
plan the API sleeps after 15 idle minutes, so the first request can take up to a minute;
the app shows a "still working" notice meanwhile.

## User roles

| Role | Can do |
|---|---|
| Visitor (not signed in) | browse, search, view places and reviews, preview recommendations |
| User | everything above, plus save trips, build and edit itineraries, review places, submit places |
| Moderator | everything above, plus approve or reject submissions and remove any review |
| Admin | reserved in the schema; currently has the same permissions as a moderator |

Login credentials for the deployed demo accounts are supplied separately with the
submission, not in this repository.

## Known limitations

- Demo places have approximate coordinates and **unknown** fees, opening hours and
  facilities, so turning on a must-have (pet, parking, wheelchair) with demo data returns
  "not enough places match". This is intended behaviour.
- With `MAP_PROVIDER=estimate`, travel times are distance-based estimates and are marked as such.
- Route ordering is a heuristic (nearest neighbour + 2-opt), not a guaranteed shortest route.
- Not yet in the web app: the AI assistant, place photos, profile editing, frontend automated tests.

## Contributing

Read [CONTRIBUTING.md](CONTRIBUTING.md). In short: branch from `main`, never commit secrets,
keep `openapi.yaml` in sync with the code, make sure `npm test` passes, and open a pull
request for review.
