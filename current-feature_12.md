# Current Feature

## Feature 012, Question analytics and retake policy

## Goal
An admin can see which questions people get wrong and where they drop off, and
retaking a quiz is no longer a memory test of the same five questions in the
same order.

## In scope
- An admin analytics page built from data already being collected
- Shuffled question and choice order, stable within an attempt
- A retake cooldown and an attempt limit
- Tests

## Out of scope
- Question banks larger than five with random selection. It is the obvious next
  step and it deserves its own feature.
- Per user analytics, cohort reporting, exports
- Charts beyond simple bars. Numbers in a table are enough to learn from.
- Changing the pass threshold

## Read this before starting
attempt_answers already holds everything the analytics need. Do not add tables
for reporting. If a query is slow, add an index.

## Shuffling, and why it needs a seed
Order must be stable within an attempt, or refreshing the page would reshuffle
mid quiz and the answered questions would move. Store a shuffle_seed on the
attempt and derive the order deterministically from it, so the same attempt
always renders the same order. A new attempt gets a new seed.

Shuffle both question order and, within each question, choice order. The
position columns stay authoritative for the admin, since an author needs a
stable view. Shuffling is a presentation concern applied at serve time.

## Data model
Add to attempts:
- shuffle_seed: integer, not null, default 0. Generated on attempt creation.

No other schema changes for analytics.

Add to lessons:
- retake_cooldown_minutes: integer, not null, default 0
- max_attempts: integer, nullable. Null means unlimited.
Both editable per lesson in the admin.

## Backend tasks
1. Migration for shuffle_seed and the two lesson columns. Inspect, upgrade,
   verify downgrade.
2. app/services/analytics.py, all read only:
   - lesson_stats(db, lesson_id): attempts started, attempts completed, pass
     rate, mean score, and completion rate
   - question_stats(db, lesson_id): per question, the number answered and the
     percentage correct, ordered by percentage correct ascending so the worst
     question is first
   - choice_distribution(db, question_id): how many times each choice was
     picked, with the correct one flagged
   - dropoff(db, lesson_id): for each question position, how many attempts
     reached it, showing where people abandon
   Write these as aggregate SQL, one query each. Do not load rows and count in
   Python.
3. app/routers/admin_analytics.py behind require_admin:
   - GET /admin/lessons/{id}/stats returning all four in one response
   Add indexes if any query is slow against a few thousand rows. Measure before
   adding.
4. app/services/quiz.py: apply the shuffle when serving the quiz for an attempt.
   The quiz delivery endpoint needs to know the attempt, so add an optional
   attempt_id query parameter. Without it, serve in authored order for preview.
   The leak rule is unchanged and the leak test must still pass.
5. app/services/attempts.py: on start_attempt, enforce the policy before
   creating a row:
   - if max_attempts is set and the viewer already has that many completed
     attempts for the lesson, refuse with 429 and a clear message
   - if a cooldown is set and their most recent completed attempt is more recent
     than the cooldown, refuse with 429 including when they may retry
   Count by user_id when signed in, and by viewer_id otherwise, reusing the
   cookie from feature 011. Note plainly in a comment that an anonymous viewer
   can clear a cookie to reset, and that this is accepted for now.
   Admins bypass both limits.
6. tests/test_analytics.py: seed a set of attempts with known answers, then
   assert each statistic comes back with the expected value. Deterministic data,
   no randomness in the test.
7. tests/test_retake_policy.py:
   - a second attempt inside the cooldown returns 429 with a retry time
   - a second attempt after the cooldown succeeds
   - exceeding max_attempts returns 429
   - a null max_attempts allows many attempts
   - an admin bypasses both
8. tests/test_shuffle.py:
   - two attempts on the same lesson produce different question orders, given
     different seeds
   - the same attempt requested twice produces identical order
   - every question and choice appears exactly once, nothing dropped or duplicated

## Frontend tasks
1. src/api/admin.js: getLessonStats(id).
2. src/pages/Admin/Stats/: route /admin/lessons/:id/stats. Four sections:
   - a summary row of attempts, completion rate, pass rate, mean score
   - a question table sorted worst first, showing percent correct, with anything
     under 40 percent visually flagged as likely a bad question
   - choice distribution per question, expandable
   - a drop off list by question position
   Handle the no data case with a plain message, not an empty table.
3. Link to it from the admin lesson list and the lesson editor.
4. src/pages/Quiz/: pass the attempt id when fetching the quiz so the shuffled
   order is used. Nothing else changes.
5. Handle the 429 from attempt creation with a readable message, including when
   a retry becomes possible, rather than a generic error.
6. Admin editor: expose retake_cooldown_minutes and max_attempts with helper
   text explaining that blank means unlimited.

## Acceptance criteria
- the stats page shows real numbers after several test attempts, and they are
  arithmetically correct against the database
- a question everyone gets wrong appears first and is flagged
- two attempts on the same lesson present the questions in different orders
- refreshing mid attempt preserves the order and the answers already given
- a retake inside the cooldown is refused with a readable message
- exceeding max_attempts is refused
- an admin can retake freely
- the leak test still passes
- pytest passes

## When done
Append an entry to CHANGELOG.md and stop.
