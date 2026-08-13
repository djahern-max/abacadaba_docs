# Current Feature

## Feature 019, Courses as the credit bearing unit

## Goal
A course is a thing a person completes and earns a certificate for. A lesson
becomes a segment inside a course rather than a standalone unit. Attempts,
certificates, and analytics all move up a level.

## In scope
- A `courses` table, with lessons as ordered members of a course
- Attempts and certificates keyed to a course instead of a lesson
- One assessment covering the whole course, drawn from its lessons' questions
- The watch gate satisfied only when every segment has been watched
- Admin authoring for courses, wrapping the existing lesson editor
- Public routes and pages reorganised around courses
- Migrations, and updating the tests that assume a lesson is the unit

## Out of scope
- Learning objectives, program level, prerequisites, field of study. Feature 020.
- The development and review chain. Feature 021.
- Credit calculation. Feature 022.
- Splitting review questions from assessment questions, and a per course pass
  threshold column. Feature 023. This feature keeps a single question type and a
  module level threshold, see "The pass threshold" below.
- Anything to do with the LLM content pipeline.

## Read this before starting
This is the largest structural change since feature 001 and it invalidates an
assumption that seven earlier features were built on: that a lesson is the unit
a person completes. It is being done now, ahead of the compliance features that
depend on it, for one reason — both the local and production databases were
emptied on 2026-08-12 and contain no rows worth preserving. Every month this
waits, it gets harder.

Because of that, the migration does not need to backfill. Do not write data
migration logic for existing lessons, attempts, or certificates; there are none.
Instead, put a comment at the top of the migration saying exactly that, with the
date, so a future reader understands why a schema change this invasive has no
backfill and does not conclude that one was forgotten.

Before running anything: confirm both databases are actually empty. If
`select count(*) from attempts` returns anything but zero, stop and say so
rather than proceeding.

## Naming
A **course** is the credit bearing program. A **lesson** stays a lesson — a
video segment inside a course. Do not rename lessons to segments, modules, or
units; the churn is large and buys nothing. Under the Standards a course is a
"program" and a lesson is roughly a "segment", but the code should use the words
the product uses.

## Data model
New `courses` table:
- id, slug (unique, indexed), title, description
- is_published: bool, default false
- thumbnail_key: string, nullable, same pattern as `lessons.thumbnail_key`
- retake_cooldown_minutes and max_attempts: moved up from lessons. These are
  program level policies; a person retakes a course, not a segment.
- created_at, updated_at

Changes to `lessons`:
- course_id: FK to courses, ondelete CASCADE, indexed, **not nullable**
- position: int, not null, unique per course, the same pattern already used for
  `questions.position` within a lesson
- drop retake_cooldown_minutes and max_attempts, now on courses
- everything else stays: duration_seconds, required_watch_ratio, video_key,
  thumbnail_key, is_published

Changes to `attempts`:
- course_id replaces lesson_id, ondelete CASCADE, indexed, not nullable

Unchanged, deliberately:
- `watch_progress` stays keyed to a lesson. A person watches segments, and the
  correctness work in feature 015 is per segment. The course level gate is
  computed from these rows, not stored.
- `questions` and `choices` stay attached to lessons.
- `attempt_answers` still points at a question, which still belongs to a lesson.
  No change is needed there and none should be made.

A lesson must belong to a course. Do not make `course_id` nullable "for
flexibility" — a nullable owner here means an orphan lesson is representable,
and every query downstream then has to handle a case that should not exist.

## The pass threshold
`PASS_THRESHOLD = 4` is a count, and it stops making sense the moment a course
has more than five questions. A three lesson course has fifteen.

Replace it with `PASS_RATIO = 0.8`, applied to the course's total question
count and rounded up, which preserves today's behaviour exactly for a five
question course (4/5) while generalising. Keep it a module level constant. It
becomes a per course column in feature 023 — do not add the column here.

## Backend tasks
1. `app/models/course.py`, plus the changes to `Lesson` and `Attempt` above.
   Then `alembic revision --autogenerate -m "add courses, move attempts to
   course"`. Inspect the generated migration closely; autogenerate handles
   added tables well and moved columns badly. Verify `downgrade -1` reverses.
2. `app/services/courses.py`: `get_by_slug`, `list_published`, and
   `get_with_lessons`. A course lists publicly only when it is published; its
   lesson list includes only published lessons, ordered by position.
3. `app/services/quiz.py`: the quiz for a course is every question from its
   published lessons, ordered by lesson position then question position. The
   existing shuffle logic applies across the whole set. The leak test must
   still pass unchanged — `is_correct` never appears in a public payload.
4. `app/services/watch.py`: `course_watch_status(db, course, identity)`
   returning per lesson progress plus a single `gate_met` boolean, true only
   when every published lesson with a duration meets its `required_watch_ratio`.
   Keep feature 015's rule intact: signed in matches on user_id only, anonymous
   on viewer_id with a null user_id.
5. `app/services/attempts.py`: `start_attempt` takes a course. The retake
   policy reads the course's cooldown and max attempts. The watch gate calls the
   new course level status. Scoring uses `PASS_RATIO`.
6. `app/services/certificates.py`: the certificate names the course, not the
   lesson. `_question_count` counts across the course.
7. `app/services/analytics.py`: the four aggregates key on course_id. Question
   level rows should carry their lesson title, so a drop off list says which
   segment a question came from.
8. `app/routers/`: public routes become
   - `GET /courses`, `GET /courses/{slug}`
   - `GET /courses/{slug}/quiz`
   - `POST /courses/{slug}/attempts`
   - `GET /courses/{slug}/lessons/{lesson_slug}` for a single segment
   and admin routes gain `/admin/courses` CRUD with lesson create/reorder inside
   a course. `GET /admin/courses/{id}/stats` replaces the lesson stats route.
   Attempt scoped routes (`/attempts/{id}/answers`, `/attempts/{id}/result`,
   `/attempts/{id}/certificate`) are unchanged — they key on the attempt.
9. `admin_content.validate_for_publish` moves to the course and gains: at least
   one lesson, every lesson has a video, every lesson has at least one question,
   and the existing per question rules. Keep returning every failure at once —
   feature 017's checklist depends on that shape.
10. Tests: this is most of the work. Every test that creates a lesson now needs
    a course around it. Add:
    - a course's quiz returns questions from all of its lessons, in order
    - the gate stays closed when one of three segments is unwatched
    - the gate opens when all three are watched
    - passing is 80% of the course's questions, verified on a course with more
      than five
    - a certificate names the course
    - deleting a course with completed attempts returns 409, as lessons did
    - the feature 015 leak test, moved to courses: two users, one browser, no
      cross contamination

## Frontend tasks
1. `/` lists courses. `CourseCard` reuses `LessonCard`'s 16:9 thumbnail shape
   and adds the segment count.
2. `/courses/:slug` is the course page: description, an ordered list of its
   lessons with per segment watch state, and a single "Take the assessment"
   button gated on the whole course.
3. `/courses/:slug/lessons/:lessonSlug` plays one segment, with previous and
   next links. Watch tracking is unchanged apart from the route it lives on.
4. The quiz, result, and certificate pages take a course slug. Their internals
   barely change; the attempt id still drives everything after the start call.
5. Admin: a course list, a course editor holding details plus an ordered lesson
   list with add and reorder, and the existing lesson editor nested inside it.
   Reuse `FileInput`, `VideoUploader`, `ThumbnailUploader`, and the publish
   checklist from features 017 and 018 rather than rewriting them.
6. Delete the old `/lessons/:slug` routes. There is no live traffic to preserve
   and a redirect layer is a maintenance cost for nobody's benefit.

## A thing to check rather than assume
The watch gate and the retake policy both used to read `lesson`. Confirm the
order of checks in `start_attempt` is still authenticate, then gate, then
policy, as feature 016 established. A signed out visitor should be told to sign
in, not told how many segments are left.

## Acceptance criteria
- `alembic upgrade head` creates courses and moves attempts; `downgrade -1`
  reverses cleanly on an empty database
- a course with three lessons shows all three on its page, in order
- the assessment button stays locked until every segment has been watched, and
  the page says which segment is outstanding
- the assessment serves all fifteen questions in lesson then position order
- passing at 12 of 15 issues a certificate naming the course
- the certificate PDF shows the course title and the course wide score
- admin can create a course, add three lessons to it, reorder them, and publish
- publishing a course with a lesson that has no video is refused, and the
  checklist says which lesson
- `/admin/courses/{id}/stats` shows question rows labelled with their lesson
- the leak test passes: `is_correct` appears nowhere in any public payload
- two users sharing a browser see independent watch progress
- pytest passes

## When done
Append an entry to CHANGELOG.md.

Then check COMPLIANCE.md. This feature is structural — it makes program level
credit possible rather than satisfying a requirement directly. Look for a
locator in docs/2026-Statement-on-Standards-for-CPE-Programs.pdf that it
genuinely meets. If there is none, write nothing to COMPLIANCE.md and say so
explicitly in your summary. Do not invent a locator to have something to record.

Then stop.
