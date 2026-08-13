# Current Feature

## Feature 004, Quiz data model and delivery

## Goal
A lesson has five multiple choice questions. The API serves them to the browser
with the correct answers stripped out, and the quiz renders read only.

## In scope
- questions and choices tables, with a migration
- A quiz endpoint that provably never leaks which choice is correct
- Seeded questions for the three existing lessons
- A quiz page that displays the questions, no answering yet
- Tests, including an explicit leak test

## Out of scope
- Selecting an answer, submitting, grading, scoring (feature 005)
- Confetti, attempts, pass or fail, certificates
- Auth, an admin UI for authoring questions
- Question shuffling, choice shuffling, timers, question banks larger than five

## Data model
Table questions:
- id: integer primary key
- lesson_id: integer, foreign key to lessons.id, not null, indexed,
  ondelete CASCADE
- prompt: text, not null
- position: integer, not null. Display order within the lesson, starting at 1.
- created_at, updated_at: timezone aware UTC, server defaults
- Unique constraint on (lesson_id, position)

Table choices:
- id: integer primary key
- question_id: integer, foreign key to questions.id, not null, indexed,
  ondelete CASCADE
- text: text, not null
- is_correct: boolean, not null, default false
- position: integer, not null. Display order within the question, starting at 1.
- created_at, updated_at: timezone aware UTC, server defaults
- Unique constraint on (question_id, position)

Relationships: Lesson has many Question, Question has many Choice, both ordered
by position, both cascading on delete.

Exactly one choice per question is correct. Enforce this in the seed script and
in a service level validation helper, not in a database constraint.

## Backend tasks
1. app/models/question.py and app/models/choice.py, SQLAlchemy 2.0 style.
   Import both in app/models/__init__.py so autogenerate sees them.
2. alembic revision --autogenerate -m "create questions and choices"
   Inspect the generated file before applying. Confirm both foreign keys and
   both unique constraints are present. Then alembic upgrade head, and verify
   downgrade -1 reverses cleanly.
3. app/schemas/quiz.py. This is the security surface, be deliberate:
   - ChoicePublic: id, text, position. It must not define is_correct at all.
     Do not rely on exclude or response_model_exclude to hide it. The field is
     simply absent from the class.
   - QuestionPublic: id, prompt, position, choices as list[ChoicePublic]
   - QuizPublic: lesson_slug, lesson_title, question_count,
     questions as list[QuestionPublic]
   All with model_config = ConfigDict(from_attributes=True).
4. app/services/quiz.py:
   - get_quiz_for_lesson(db, slug) returning the lesson's questions with choices
     eagerly loaded and ordered by position, or None if the lesson is missing,
     unpublished, or has no questions.
   - validate_question(question) raising a clear error unless exactly one of its
     choices is correct. Used by the seed script.
5. app/routers/quiz.py: GET /lessons/{slug}/quiz
   - 404 with a clear detail when the lesson is missing or unpublished
   - 404 with detail "This lesson has no quiz yet" when it has no questions
   - otherwise 200 with QuizPublic
   Register under /api/v1.
6. Extend backend/scripts/seed.py: five questions per seeded lesson, four
   choices each, exactly one correct. Write real questions that match each
   lesson's subject rather than filler text. Keep it idempotent, matching on
   (lesson_id, position), and print a summary. Run validate_question on every
   question before committing.
7. tests/test_quiz.py:
   - the quiz endpoint returns 200 with five questions, each having four choices
   - LEAK TEST: take response.text, the raw JSON string, and assert that
     "is_correct" does not appear anywhere in it, and that neither "true" nor
     "correct" appears as a value. Comment this test clearly as the guard that
     must never be deleted.
   - questions come back ordered by position, and so do choices
   - a lesson with no questions returns 404
   - an unknown slug returns 404

## Frontend tasks
1. src/api/quiz.js: getQuiz(slug).
2. src/pages/Quiz/: route "/lessons/:slug/quiz". Fetch on mount. Render the
   lesson title, "5 questions", and every question in order with its choices as
   plain unselectable rows labeled A, B, C, D. Handle loading, error, and the
   no-quiz-yet 404 with a friendly message and a link back to the lesson.
   Add a visible note that answering arrives next, so the page does not look
   broken.
3. Register the route in App.jsx.
4. src/pages/LessonDetail/: add a "Take the quiz" button linking to the quiz
   route. Show it only when the lesson has a quiz. Rather than add a field to
   LessonDetail, simply always show the button and let the quiz page handle the
   404 case gracefully.
5. src/components/QuestionCard/ with its own CSS Module, used by the quiz page,
   built so feature 005 can add selection state without a rewrite.

## Acceptance criteria
- alembic upgrade head creates both tables, downgrade -1 reverses them
- python -m scripts.seed adds five questions per lesson and is still idempotent
  on a second run
- curl on /api/v1/lessons/intro-to-ratios/quiz returns five questions with four
  choices each
- piping that response through grep -i correct finds nothing
- the quiz page renders all five questions in order at
  localhost:5173/lessons/intro-to-ratios/quiz
- pytest passes, including the leak test

## When done
Append an entry to CHANGELOG.md and stop.
