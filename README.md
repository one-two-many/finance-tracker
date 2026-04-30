# Finance Tracker

A self-hosted personal finance tracker. FastAPI backend + React/TypeScript frontend, PostgreSQL for storage, all wired up with Docker Compose.

Import CSVs from your bank/credit cards (Amex, Chase, Discover, Capital One, Citi, Bank of America, Wells Fargo), auto-categorize transactions, visualize money flow with a Sankey diagram, and split expenses with friends via Splitwise.

## Features

- Multi-bank CSV import with auto-detect (and a PDF parser for Chase statements)
- Auto-categorization with user-defined rules
- Transfer detection between your accounts
- Hierarchical categories with colors and icons
- Sankey money-flow visualization
- Net worth tracking with balance snapshots
- Households / shared accounts
- Splitwise integration (encrypted API-key storage, group/friend splits)
- JWT authentication, bcrypt password hashing
- OpenTelemetry traces/metrics/logs (optional)

## Tech Stack

**Backend** — FastAPI, SQLAlchemy, PostgreSQL, Pydantic, python-jose, passlib, splitwise SDK, OpenTelemetry
**Frontend** — React 18, TypeScript, Vite, Tailwind CSS, shadcn/ui, recharts
**Infra** — Docker Compose, GitHub Actions → GHCR

---

## Run Locally with Docker (recommended)

This is the fastest path. Everything runs in containers — no Python or Node needed on the host.

### Prerequisites
- Docker Desktop (or Docker Engine + Compose v2)
- Git (with submodule support)

### Steps

```bash
# 1. Clone with submodules
git clone --recurse-submodules https://github.com/one-two-many/finance-tracker.git
cd finance-tracker

# If you already cloned without --recurse-submodules:
# git submodule update --init --recursive

# 2. Create your .env from the template
cp .env.example .env

# 3. Generate a SECRET_KEY and put it in .env
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
# Open .env, paste the value into SECRET_KEY=, and set a strong POSTGRES_PASSWORD.

# 4. Start everything
docker-compose up -d

# 5. Watch logs (optional)
docker-compose logs -f
```

The first start pulls the prebuilt images from GHCR and runs database migrations automatically. Once it's up:

| Service       | URL                                |
|---------------|------------------------------------|
| Frontend      | http://localhost:3000              |
| Backend API   | http://localhost:8000              |
| API docs      | http://localhost:8000/docs         |
| Health check  | http://localhost:8000/health       |

Register a new user from the frontend's sign-up page and start importing CSVs.

### Stopping / resetting

```bash
docker-compose down            # stop containers, keep data
docker-compose down -v         # stop AND wipe the database (irreversible)
docker-compose pull            # update to latest published images
docker-compose up -d --force-recreate
```

### Required environment variables

`docker-compose.yml` will refuse to start if any of these are missing:

| Variable            | What it's for                                                              |
|---------------------|----------------------------------------------------------------------------|
| `SECRET_KEY`        | JWT signing + Fernet key for encrypting Splitwise API keys at rest         |
| `POSTGRES_USER`     | Postgres username                                                          |
| `POSTGRES_PASSWORD` | Postgres password                                                          |
| `POSTGRES_DB`       | Postgres database name                                                     |

Optional:

| Variable                       | Default                              | Notes                                              |
|--------------------------------|--------------------------------------|----------------------------------------------------|
| `ALLOWED_ORIGINS`              | `localhost:3000,localhost:5173`      | Comma-separated CORS allow-list                    |
| `OTEL_EXPORTER_OTLP_ENDPOINT`  | _(empty — disables OTel export)_     | Backend OTLP/HTTP collector URL                    |
| `VITE_OTEL_ENDPOINT`           | _(empty — disables OTel export)_     | Frontend OTel endpoint (web telemetry)             |

---

## Run Locally for Development

Use this if you want hot-reload and to edit the code. Postgres still runs in Docker; backend and frontend run on the host.

### Prerequisites
- Python 3.11+
- Node.js 20+ (npm 10+)
- Docker Desktop (for Postgres)

### 1. Start just the database

```bash
docker-compose -f docker-compose-dev.yml up -d
```

This brings up Postgres on `localhost:5432` with the dev defaults (`financeuser` / `financepass` / `financedb`). Override with env vars or a `.env` if you want different ones.

### 2. Backend

```bash
cd finance-tracker-backend
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Create `finance-tracker-backend/.env`:

```env
FT_DATABASE_URL=postgresql://financeuser:financepass@localhost:5432/financedb
SECRET_KEY=<paste a 32-byte random secret here>
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5173
DEBUG=True
```

Run migrations once (the Docker build does this automatically; for host-mode you do it manually):

```bash
DB_HOST=localhost DB_PORT=5432 \
DB_USER=financeuser DB_NAME=financedb POSTGRES_PASSWORD=financepass \
bash scripts/run_migrations.sh
```

Start the API:

```bash
python3 -m uvicorn app.main:app --reload
```

Backend is now on http://localhost:8000 (docs at `/docs`).

### 3. Frontend

```bash
cd finance-tracker-frontend
npm install
```

Create `finance-tracker-frontend/.env`:

```env
VITE_API_URL=http://localhost:8000
```

Start the dev server:

```bash
npm run dev
```

Frontend is on http://localhost:5173 with HMR.

---

## Common Tasks

```bash
# Tail backend logs
docker-compose logs -f backend

# Open a Postgres shell
docker exec -it finance-tracker-db psql -U $POSTGRES_USER -d $POSTGRES_DB

# Check applied migrations
docker exec -it finance-tracker-db psql -U $POSTGRES_USER -d $POSTGRES_DB -c "SELECT * FROM schema_migrations;"

# Backup the database
docker exec finance-tracker-db pg_dump -U $POSTGRES_USER $POSTGRES_DB > backup.sql

# Restore the database
docker exec -i finance-tracker-db psql -U $POSTGRES_USER -d $POSTGRES_DB < backup.sql

# Type-check the frontend
cd finance-tracker-frontend && npx tsc --noEmit

# Lint the frontend
cd finance-tracker-frontend && npm run lint
```

---

## Repository Layout

This repo is a meta-repo that pins two submodules:

```
finance-tracker/                  ← this repo (compose, env, docs)
├── docker-compose.yml            ← prod-style compose using prebuilt images
├── docker-compose-dev.yml        ← Postgres only, for host-mode dev
├── .env.example                  ← copy to .env
├── finance-tracker-backend/      ← submodule: FastAPI service
└── finance-tracker-frontend/     ← submodule: React app
```

Pull submodule updates with `git submodule update --remote --merge`.

---

## Troubleshooting

**`docker-compose up` fails with "POSTGRES_USER must be set in .env"**
You haven't created `.env` yet, or it's missing required keys. Run `cp .env.example .env` and fill it in.

**Frontend shows "Network Error"**
Backend isn't running or CORS is wrong. Check `curl http://localhost:8000/health` and verify `ALLOWED_ORIGINS` in `.env` includes your frontend origin.

**"No accounts found" when importing CSVs**
Create an account first via the dashboard's "+ Create Account" button.

**Sankey page crashes with "Maximum call stack size exceeded"**
Indicates a cycle in the graph data. The backend already guards against income/expense category-name collisions; if you see this, check `app/services/sankey_service.py`.

---

## License

MIT
