# RemoteLED

> **Git policy — never auto-commit or auto-push.** Leave your work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or `scripts/push_all.sh` **only when the user
> explicitly asks in that turn**. Approval for an earlier change does not carry forward, and
> finishing a task is not permission to commit it.

QR-to-device payment activation system. Customers scan a QR code on a Raspberry Pi kiosk, select a service (laundry, vending, air compressor), pay via Stripe, and the Pi activates a GPIO relay for the authorized duration. The backend signs ECDSA authorizations; the Android app relays them to the Pi over BLE.

## Monorepo Structure

```
backend/             Python FastAPI API server (port 9999)
  app/
    api/             Route modules: admin, auth, authorizations, devices, led, orders, payments, reference, telemetry
    core/            Config, database, auth, LED handler, payment handler, validators, admin logger
    models/          Pydantic request/response schemas
    services/        ECDSA crypto signing
frontend/
  admin-react/       React 18 + TypeScript + Vite admin console, served at /admin/ (dev port 5173)
    src/
      main.tsx       Entry: BrowserRouter basename=/admin + AuthProvider (contexts/AuthContext)
      App.tsx        Routes -> pages/Dashboard, pages/Login
      pages/         ACTIVE route pages (Dashboard, Login) — what the app actually renders
      hooks/         ACTIVE data hooks (use*.ts) used by pages/Dashboard
      api/           ACTIVE fetch-based apiClient + per-resource modules
      components/    ACTIVE UI: common/, dashboard/, forms/, layout/ (+ flat Badge/Button/etc)
      contexts/      ACTIVE AuthContext (localStorage JWT, key `remoteled_auth_token`)
      types/ config/ constants/ utils/   ACTIVE shared code
      core/          Parallel refactor (Axios client, types, utils) — NOT wired into App.tsx
      features/      Parallel refactor (auth/dashboard/devices/orders/products) — NOT wired in
      context/       Singular dir — unused alt AuthContext/AppContext
database/            PostgreSQL schema, seed data, migrations
pi/                  Raspberry Pi BLE peripheral (Python; see gotchas) + kiosk (static web/ and CRA kiosk/)
android/             Kotlin Android app (BLE GATT + QR scanner), minSdk 26
tests/               Shell-script integration tests
docs/                Architecture docs (ARCHITECTURE.md, SYSTEM.md)
plans/               Improvement plans
```

## Tech Stack

| Component | Stack |
|-----------|-------|
| Backend | Python 3.11+, FastAPI, psycopg2 (sync), Pydantic, Stripe, ECDSA (secp256k1), Bleak (BLE) |
| Frontend | React 18, TypeScript 5.5, Vite 5, Recharts, React Router 6. Active client uses `fetch`; the unused `core/` refactor uses Axios |
| Database | PostgreSQL 15+, raw SQL with RealDictCursor (no ORM) |
| Pi | Python (bluezero) BLE peripheral, nginx kiosk (Node variant is not shipped — `pi/node/main.js` missing) |
| Android | Kotlin, BLE GATT, CameraX + ML Kit QR, minSdk 26 |
| Packages | uv (Python), npm (Node), Gradle (Android) |

## Key Commands

```bash
# Backend
cd backend && uv run --no-project uvicorn app.main:app --reload --host 0.0.0.0 --port 9999
# Swagger docs: http://localhost:9999/docs

# Frontend
cd frontend/admin-react && npm install        # axios/tailwindcss are imported/configured but not in package.json
cd frontend/admin-react && npm run dev         # http://localhost:5173/admin/
cd frontend/admin-react && npm run test:unit   # vitest (NOTE: script is test:unit; `npm run test` / `make test-frontend` fail)

# Database
createdb remoteled
psql -d remoteled -f database/schema.sql
psql -d remoteled -f database/seed.sql

# Docker (full admin stack)
docker compose -f docker-compose.admin.yml up

# Makefile shortcuts
make dev-backend      # start FastAPI (port 9999)
make dev-frontend     # start Vite
make db-reset         # drop + recreate + seed
make install          # uv sync + npm install
# make test / make test-frontend are BROKEN: they call `npm run test`, which doesn't exist (use test:unit)
# make test-backend runs `uv run pytest` but pytest is only in backend/requirements.txt, not pyproject.toml
```

## Database Schema

### Enums
- `service_type`: TRIGGER | FIXED | VARIABLE
- `order_status`: CREATED -> PAID -> RUNNING -> DONE (or FAILED at any step)
- `device_status`: ACTIVE | OFFLINE | MAINTENANCE | DEACTIVATED
- `log_direction`: PI_TO_SRV | SRV_TO_PI

### Tables
| Table | Purpose | Key Columns |
|-------|---------|-------------|
| admins | Admin users | email, password_hash, role |
| devices | Pi devices | label, public_key, model_id FK, location_id FK, gpio_pin, status |
| device_models | Reference: hardware models | name |
| locations | Reference: physical locations | name |
| service_types | Reference: enum labels | name, code (service_type enum) |
| services | Products/pricing | type, price_cents, fixed_minutes, minutes_per_25c, active |
| device_services | M2M: device <-> service | device_id FK, service_id FK (unique pair) |
| orders | Customer orders | device_id FK, service_id FK, amount_cents, authorized_minutes, status |
| authorizations | Signed payloads | order_id FK (unique), device_id FK, payload_json, signature_hex, expires_at |
| logs | Device telemetry | device_id FK, direction, ok, details |
| admin_logs | Audit trail | admin_id FK, action, entity_type, entity_id |

### Views
- `v_devices_summary` — devices with service counts and order stats
- `v_orders_detailed` — orders joined with device/service info
- `v_logs_recent` — last 100 logs with device labels

### Functions
- `get_device_services(device_uuid)` — active services for a device
- `calculate_variable_minutes(service_uuid, amount_cents)` — compute authorized time

## API Route Structure

| Prefix | Module | Auth | Purpose |
|--------|--------|------|---------|
| /auth | auth.py | No | Register, login, logout, me |
| /devices | devices.py | No | Public device + service lookup |
| /orders | orders.py | No | Order CRUD + status transitions |
| /authorizations | authorizations.py | No | ECDSA authorization create/read |
| /payments | payments.py | No | Stripe payment + LED trigger |
| /led | led.py | No | Direct BLE LED control |
| /admin | admin.py | JWT | Stats, device/service/order CRUD, logs |
| /admin | reference.py | JWT | Device models, locations, service types CRUD |
| /devices | telemetry.py | No | Device event logging |

## Architecture Patterns

### Backend
- **Database**: global `db` singleton (`app/core/database.py`); `get_db()` yields a psycopg2 `RealDictCursor` via FastAPI `Depends()`. Connection auto-commits on exit, rollbacks on exception.
- **SQL**: raw SQL everywhere with `%s` parameterization. Results are dicts. No ORM.
- **Auth**: JWT via python-jose (HS256), bcrypt passwords. `Depends(get_current_user)` for protected endpoints. Token in `Authorization: Bearer <token>`.
- **Validation**: Pydantic models for request/response. UUID validation via `app/core/validators.py`.
- **Admin logging**: `log_admin_action()` uses its own DB connection (won't fail main transaction).
- **Crypto**: ECDSA secp256k1 signing in `app/services/crypto.py`.
- **BLE**: Bleak library for macOS/Linux BLE communication in `app/core/led_handler.py`.

### Frontend (what actually renders)
- **Entry**: `main.tsx` -> `App.tsx` -> `pages/Dashboard.tsx` + `pages/Login.tsx`. This is the live tree. `features/` + `core/` are an incomplete parallel refactor App.tsx never imports — add new code to the active path (`pages/`, `hooks/`, `components/`, `api/`), not to `features/`/`core/`.
- **API client**: `src/api/client.ts` — a thin `fetch` wrapper (`apiClient.get/post/...`). Base URL from `src/config/constants.ts`.
- **Auth**: `src/contexts/AuthContext.tsx` (the one `main.tsx` uses). Token in localStorage as `remoteled_auth_token` (NOT `access_token` — that key belongs to the unused `core/api/client.ts`).
- **Hooks**: `src/hooks/use*.ts` (useState + useCallback + try/catch). `pages/Dashboard.tsx` consumes these.
- **Routing**: React Router 6. Base path `/admin/`. Vite proxy: `/api` -> `http://localhost:9999`.
- **Styling**: `index.css` has `@tailwind` directives and `tailwind.config.js` exists, but `tailwindcss` is not in package.json and `postcss.config.js` only loads autoprefixer — Tailwind is effectively not wired; styling leans on custom CSS in `App.css`/`index.css`.

## Common Pitfalls

- **Port 9999**: Backend runs on 9999 locally. Docker compose uses 8000. Do NOT hardcode ports in frontend — use the API client.
- **No ORM**: Database is raw psycopg2. Do not introduce async DB calls (asyncpg) or SQLAlchemy without migrating everything.
- **Dual frontend paths**: the ACTIVE path is `pages/` + `hooks/` + `components/` + `api/` (fetch) + `contexts/AuthContext`. `core/` (Axios) + `features/` + `context/` (singular) are an unfinished parallel refactor that `App.tsx`/`main.tsx` never import. Add new code to the active path; do not assume `features/`/`core/` is live.
- **Firebase is dead**: `firebase-admin` is in pyproject.toml and `firebase` is in frontend package.json, but neither is used. Auth is local JWT. Do not re-add Firebase.
- **No prebuilt APK / missing Pi files**: no `remoteled.apk` is committed; `pi/node/main.js` and the `pi/kiosk.sh` that `pi/install.sh` copies are both absent — only the Python BLE path works, and the installer needs a `kiosk.sh` supplied.
- **Service type constraints**: TRIGGER must not have fixed_minutes/minutes_per_25c. FIXED requires fixed_minutes. VARIABLE requires minutes_per_25c. Enforced at both DB and API level.
- **Order status transitions**: Strictly validated via `valid_transitions` dict in `orders.py`. Cannot skip states.
- **UUID validation**: All IDs are UUID v4. Use `validate_uuid()` from `app/core/validators.py`.
- **Admin logger**: Opens its own DB connection (separate from request cursor). This is intentional.
- **Connection per request**: psycopg2 creates a new connection per request (no pool). This is a known limitation.

## Environment Setup

```bash
# Prerequisites: Python 3.11+, Node.js 18+, PostgreSQL 15+, uv

# Install dependencies
uv sync                                    # Python
cd frontend/admin-react && npm install     # Frontend

# Database
createdb remoteled
psql -d remoteled -f database/schema.sql
psql -d remoteled -f database/seed.sql

# Configure backend
cp backend/.env.example backend/.env       # Edit with your Stripe keys if needed

# Run
cd backend && uv run --no-project uvicorn app.main:app --reload --host 0.0.0.0 --port 9999
cd frontend/admin-react && npm run dev
```

## Git Workflow

- Do not use feature branches for code changes. Always stay on the current branch.
