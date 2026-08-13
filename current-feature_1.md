# Current Feature

## Feature 001, Walking skeleton

## Goal
Prove the whole chain works end to end: Postgres, FastAPI, and the React app all
talking to each other. No product features yet.

## In scope
- Backend config, database session, health endpoint, Alembic set up and runnable
- Frontend stripped of Vite boilerplate, global CSS variables, one status banner
- One backend test

## Out of scope
- Any lesson, video, quiz, user, or certificate code
- Auth, DigitalOcean Spaces, deployment

## Backend tasks
1. Create backend/.venv, install backend/requirements.txt.
2. app/config.py: Settings via pydantic-settings reading DATABASE_URL and
   CORS_ORIGINS from backend/.env. CORS_ORIGINS is a comma separated string
   parsed into a list.
3. app/db.py: engine from DATABASE_URL, SessionLocal, declarative Base,
   get_db dependency that yields a session and always closes it.
4. app/main.py: create the FastAPI app titled "abacadaba API", add CORS
   middleware from settings, include the health router under prefix /api/v1.
5. app/routers/health.py: GET /health that executes SELECT 1 against the
   database and returns {"status": "ok", "database": "connected"}. If the query
   fails, return 503 with {"status": "error", "database": "unavailable"}.
6. app/schemas/health.py: HealthResponse schema used as the response_model.
7. Run alembic init alembic inside backend/. Edit alembic/env.py to read the URL
   from app.config settings rather than alembic.ini, and set
   target_metadata = Base.metadata. Leave alembic.ini's sqlalchemy.url empty.
8. tests/test_health.py: use httpx and FastAPI's TestClient to assert
   GET /api/v1/health returns 200 and status "ok".

## Frontend tasks
1. Delete the Vite boilerplate: App.css, the logo assets, and the counter demo.
2. src/styles/global.css: CSS custom properties for colors, spacing, radius, and
   font stack. Pick a friendly, high contrast palette with one accent color.
   Import it once in main.jsx.
3. src/api/client.js: a small helper that prefixes import.meta.env.VITE_API_URL,
   sets JSON headers, throws on non-2xx responses, and returns parsed JSON.
4. src/api/health.js: getHealth() using the client helper.
5. src/App.jsx plus src/App.module.css: on mount, call getHealth() and render the
   app name with a status pill reading "Backend connected" in green or
   "Backend unreachable" in red.

## Acceptance criteria
- docker compose up -d brings Postgres up on 5432
- uvicorn app.main:app --reload starts with no errors
- curl http://localhost:8000/api/v1/health returns status ok, database connected
- alembic upgrade head runs cleanly (no revisions yet is fine)
- npm run dev renders the green "Backend connected" pill at localhost:5173
- pytest passes

## When done
Append an entry to CHANGELOG.md and stop.