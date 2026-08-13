# Backend, FastAPI

## Layout
    app/main.py      app creation, CORS, router registration
    app/config.py    pydantic-settings Settings loaded from .env
    app/db.py        engine, SessionLocal, Base, get_db dependency
    app/models/      SQLAlchemy models, one file per table
    app/schemas/     Pydantic v2 request and response models
    app/routers/     thin HTTP layer, one file per resource
    app/services/    business logic: quiz scoring, Spaces client, certificates
    alembic/         migrations
    tests/           pytest

## Rules
- Routers stay thin: validate input, call a service, return a schema. No SQL in routers.
- Use SQLAlchemy 2.0 style: Mapped, mapped_column, select().
- Never return ORM objects directly. Everything goes through a Pydantic schema.
- Sessions come from Depends(get_db). Services receive a session, they never create one.
- Table names are plural snake_case. Primary key is an integer id unless stated otherwise.
- Timestamps are timezone-aware UTC: DateTime(timezone=True).
- Raise HTTPException with a clear detail message rather than letting a 500 escape.
- Security rule that never bends: correct answers are never included in any response
  sent before grading. Grading happens on the server.
- Model change means: alembic revision --autogenerate -m "..." then alembic upgrade head.

## Env vars
DATABASE_URL, CORS_ORIGINS, SPACES_KEY, SPACES_SECRET, SPACES_REGION,
SPACES_BUCKET, SPACES_ENDPOINT
EOF