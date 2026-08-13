# abacadaba.com

Micro-learning. A user watches a ~5 minute video, takes a 5-question multiple
choice quiz, gets a small burst of confetti for each correct answer, and at 4/5
or better passes: bigger confetti plus a downloadable certificate.
The name is a play on guessing your way through a multiple choice test.

## Stack
- Backend: Python 3.12, FastAPI, SQLAlchemy 2.0, Alembic, Postgres 16
- Frontend: React 18 + Vite, plain JavaScript (no TypeScript), CSS Modules
- Video storage: DigitalOcean Spaces (S3-compatible) via boto3, served as presigned URLs
- Local Postgres runs in Docker

## Layout
    backend/            FastAPI app (see backend/CLAUDE.md)
    frontend/           Vite React app (see frontend/CLAUDE.md)
    current-feature.md  the ONE feature being built right now
    CHANGELOG.md        completed features, append only

## Workflow, read this first
1. current-feature.md is the single source of truth. Build exactly what it describes.
2. Do not build anything outside the feature's scope. If you hit something needed
   but out of scope, finish the feature and list it at the end of your response.
3. When every acceptance criterion passes, append a short entry to CHANGELOG.md
   (date, feature number, one or two lines on what shipped) and say the feature is done.
4. Never rewrite CHANGELOG.md history. Append only.

## Conventions
- snake_case in Python and in the database, camelCase in JS, PascalCase for components.
- API routes live under /api/v1.
- Every model change ships with its Alembic migration in the same change.
- Secrets go in .env, never committed. Add new vars to .env.example too.
- Prefer small readable code over clever code. Justify any new dependency.

## Commands
    docker compose up -d                                  # Postgres
    cd backend && source .venv/bin/activate && uvicorn app.main:app --reload
    cd frontend && npm run dev
    cd backend && pytest