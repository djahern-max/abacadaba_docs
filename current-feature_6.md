# Current Feature

## Feature 006, Attempts, scoring, and passing at 4 of 5

## Goal
Taking a quiz becomes a recorded attempt. The server stores every answer,
refuses to let a question be answered twice, computes the score itself, and
decides pass or fail at 4 of 5. Passing fires the big confetti.

## In scope
- attempts and attempt_answers tables, with a migration
- Endpoints to start an attempt, answer within it, and read the result
- Server computed score and pass or fail
- A results screen with big confetti on a pass
- Replacing the sessionless grading endpoint from feature 005
- Tests, including the replay protection this feature exists to add

## Out of scope
- Certificates and PDF generation (feature 007)
- Auth and user accounts (feature 008). Attempts are anonymous for now.
- Attempt history, a "my results" page, analytics
- Time limits, question shuffling, partial credit

## The security change this feature makes
Feature 005 graded each answer with no server side session, so the correct
choice could be discovered by resubmitting. Attempts fix that: each answer is
recorded against an attempt, and a question already answered in that attempt is
rejected with a 409. Remove the old POST /lessons/{slug}/quiz/answers endpoint
and its tests entirely rather than leaving it alongside the new path. Delete the
limitation comment it carried.

## Data model
Table attempts:
- id: integer primary key
- public_id: UUID, unique, indexed, not null, generated server side. This is
  what appears in URLs and request bodies. The integer id is never exposed.
- lesson_id: integer, foreign key to lessons.id, not null, indexed,
  ondelete CASCADE
- score: integer, nullable. Null until the attempt is complete.
- passed: boolean, nullable. Null until complete.
- completed_at: timezone aware UTC, nullable
- created_at, updated_at: timezone aware UTC, server defaults

Table attempt_answers:
- id: integer primary key
- attempt_id: integer, foreign key to attempts.id, not null, indexed,
  ondelete CASCADE
- question_id: integer, foreign key to questions.id, not null, indexed
- choice_id: integer, foreign key to choices.id, not null
- is_correct: boolean, not null. Recorded at answer time.
- created_at: timezone aware UTC, server default
- Unique constraint on (attempt_id, question_id). This constraint is the
  replay protection. It must exist in the database, not only in service code.

Pass mark: 4 of 5. Define it as a module level constant PASS_THRESHOLD in the
quiz service and reference it everywhere rather than writing 4 inline.

## Backend tasks
1. app/models/attempt.py and app/models/attempt_answer.py, SQLAlchemy 2.0 style.
   Import both in app/models/__init__.py. Use uuid4 as the default for public_id.
2. alembic revision --autogenerate -m "create attempts and attempt answers"
   Inspect the file before applying, confirming the unique constraint and all
   foreign keys are present. Then upgrade head, and verify downgrade -1 reverses.
3. app/schemas/attempt.py:
   - AttemptStart: attempt_id as the UUID, lesson_slug, question_count
   - AttemptAnswerRequest: question_id, choice_id
   - AttemptAnswerResponse: correct, correct_choice_id, answered_count,
     question_count
   - AttemptResult: attempt_id, lesson_slug, lesson_title, score,
     question_count, passed, completed_at
4. app/services/attempts.py:
   - start_attempt(db, slug) creating a row for a published lesson that has
     questions. Returns None if the lesson is missing, unpublished, or quizless.
   - record_answer(db, public_id, question_id, choice_id) which:
     - loads the attempt by public_id, or signals not found
     - refuses with a conflict signal if the attempt is already complete
     - confirms the question belongs to the attempt's lesson
     - confirms the choice belongs to the question
     - catches the unique constraint violation on (attempt_id, question_id) and
       turns it into a conflict signal, so a race cannot slip past
     - stores is_correct and returns the result plus the running answered count
     - when the answered count reaches question_count, computes score from the
       stored answers, sets passed using PASS_THRESHOLD, stamps completed_at,
       and commits
   - get_result(db, public_id) returning AttemptResult data, or a signal that
     the attempt is not complete yet.
   Score is always computed by counting attempt_answers rows where is_correct is
   true. Never trust a count sent by the client.
5. app/routers/attempts.py:
   - POST /lessons/{slug}/attempts, 201 with AttemptStart, 404 if unavailable
   - POST /attempts/{attempt_id}/answers, 200 with AttemptAnswerResponse,
     404 unknown attempt, 409 if already answered or the attempt is complete,
     400 if the choice does not belong to the question
   - GET /attempts/{attempt_id}/result, 200 with AttemptResult,
     404 unknown, 409 if not yet complete
   Register under /api/v1. Delete the old answers route.
6. tests/test_attempts.py:
   - starting an attempt returns a UUID and question_count 5
   - answering all five correctly gives score 5 and passed true
   - answering four correctly gives score 4 and passed true, the boundary
   - answering three correctly gives score 3 and passed false, the other side
   - REPLAY TEST: answering the same question twice in one attempt returns 409.
     Comment this clearly as the guard that replaced the feature 005 hole.
   - answering after the attempt is complete returns 409
   - reading a result before completion returns 409
   - an unknown attempt id returns 404
   - a choice from another question returns 400
   Remove the tests for the deleted endpoint.

## Frontend tasks
1. src/api/attempts.js: startAttempt(slug), submitAttemptAnswer(attemptId,
   questionId, choiceId), getAttemptResult(attemptId). Remove submitAnswer from
   src/api/quiz.js.
2. src/lib/confetti.js: implement bigBurst() properly, replacing the stub.
   Something clearly more celebratory than smallBurst: a wider spread, more
   particles, and two or three staggered volleys over roughly two seconds. Still
   a no-op under prefers-reduced-motion.
3. src/pages/Quiz/: start an attempt when the quiz loads, hold the attempt id in
   state, and send answers to the attempt endpoints. The per question flow and
   the small burst are unchanged. Replace the "Quiz complete" placeholder with a
   redirect to the results route once the final answer comes back.
4. src/pages/Result/: route "/attempts/:attemptId". Fetch the result on mount
   and render the lesson title, the score as "4 out of 5", and a clear pass or
   fail state.
   - On a pass: a congratulatory heading, fire bigBurst() once on mount, and a
     disabled placeholder button reading "Download certificate" with a note that
     it arrives next. Do not fire the burst again on re-render.
   - On a fail: an encouraging message, the score, and a "Watch again" link back
     to the lesson plus a "Retry quiz" link that starts a fresh attempt.
   - Handle the 409 not yet complete case with a message and a link home rather
     than an error screen.
   The result route is reachable by URL, so a refresh must still work.
5. Register the route in App.jsx.
6. Accessibility: the pass or fail outcome is conveyed in text, not by color
   alone, and the heading receives focus on mount so a screen reader announces
   the result.

## Acceptance criteria
- alembic upgrade head creates both tables, downgrade -1 reverses them
- curl POST to start an attempt returns a UUID
- curl answering the same question twice in that attempt returns 409
- curl answering four of five correctly returns a result with score 4 and
  passed true
- curl answering three of five returns passed false
- GET on the result before finishing returns 409
- the old /quiz/answers endpoint is gone and returns 404
- completing a quiz in the browser lands on /attempts/<uuid> and refreshing that
  URL still shows the result
- passing fires a noticeably larger confetti celebration than a single correct
  answer
- failing shows the retry and rewatch links, no confetti
- pytest passes, including the leak test and the new replay test

## When done
Append an entry to CHANGELOG.md and stop.
