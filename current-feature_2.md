# Current Feature

## Feature 002, Lessons: model, API, and browsing

## Goal
A real domain object. Seeded lessons come out of Postgres, through the API, and
render as a browsable list with a detail page.

## In scope
- lessons table with an Alembic migration
- List and detail endpoints
- A seed script with three sample lessons
- Client side routing, list page, detail page
- Backend tests for the new endpoints

## Out of scope
- Video upload, DigitalOcean Spaces, any video playback (feature 003)
- Questions, quizzes, scoring, confetti, certificates
- Auth, admin UI, creating or editing lessons through the API
- Pagination, search, filtering

## Data model
Table lessons:
- id: integer primary key
- slug: string, unique, indexed, not null. URL safe, e.g. "intro-to-ratios"
- title: string, not null
- description: text, not null
- duration_seconds: integer, nullable
- is_published: boolean, not null, default false
- created_at, updated_at: timezone aware UTC, server defaults

Do not add a video column. That arrives in feature 003 with its own migration.

## Backend tasks
1. app/models/lesson.py: the Lesson model in SQLAlchemy 2.0 style.
2. Make sure app/models/__init__.py imports Lesson, and that alembic/env.py
   imports app.models so autogenerate actually sees the table. Verify the
   generated migration is not empty before applying it.
3. Generate and apply the migration:
   alembic revision --autogenerate -m "create lessons"
   alembic upgrade head
4. app/schemas/lesson.py: LessonSummary (id, slug, title, description,
   duration_seconds) and LessonDetail (same fields for now, kept separate so it
   can grow in later features). model_config = ConfigDict(from_attributes=True).
5. app/services/lessons.py: list_published(db) ordered by id, and
   get_by_slug(db, slug) returning None when missing or unpublished.
6. app/routers/lessons.py:
   - GET /lessons returns list[LessonSummary], published only
   - GET /lessons/{slug} returns LessonDetail, or 404 with a clear detail message
   Register under the /api/v1 prefix in main.py.
7. backend/scripts/seed.py: idempotent, safe to run twice. Upsert three
   published lessons on slug with plausible titles and two or three sentence
   descriptions, duration_seconds around 300. Print what it created or skipped.
   Runnable as: python -m scripts.seed
8. tests/test_lessons.py:
   - GET /api/v1/lessons returns 200 and a list
   - an unpublished lesson does not appear in the list
   - GET /api/v1/lessons/{slug} returns the right lesson
   - an unknown slug returns 404

## Frontend tasks
1. npm install react-router-dom. This is the one new dependency; two routes need
   real URLs so a lesson can be linked and refreshed.
2. src/api/lessons.js: getLessons() and getLesson(slug) via the existing client.
3. src/main.jsx: wrap App in BrowserRouter.
4. src/App.jsx: an app shell with a header showing the abacadaba wordmark and the
   existing health pill, plus Routes:
   - "/" renders LessonList
   - "/lessons/:slug" renders LessonDetail
   - "*" renders a small not found message with a link home
5. src/components/LessonCard/: title, description, and duration formatted as
   "5 min". The whole card is a Link to /lessons/:slug. Hover state.
6. src/pages/LessonList/: fetch on mount, render a responsive grid of
   LessonCards. Handle three states explicitly: loading, error, empty.
7. src/pages/LessonDetail/: fetch by slug from useParams. Render title,
   duration, description, a back link to "/", and a placeholder box reading
   "Video coming soon" sized at 16:9. Show a friendly message on 404 rather
   than a blank screen.
8. Every page and component gets its own CSS Module using the custom properties
   already in global.css. Add new properties there rather than hardcoding values.

## Acceptance criteria
- alembic upgrade head creates the lessons table, alembic downgrade -1 reverses it
- python -m scripts.seed inserts three lessons, and running it twice is harmless
- curl http://localhost:8000/api/v1/lessons returns the three seeded lessons
- curl http://localhost:8000/api/v1/lessons/unknown-slug returns 404
- localhost:5173 shows three lesson cards
- clicking a card routes to /lessons/<slug> and the page survives a browser refresh
- the 16:9 placeholder is visible on the detail page
- pytest passes

## When done
Append an entry to CHANGELOG.md and stop.
