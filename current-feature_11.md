# Current Feature

## Feature 011, Watch tracking and quiz gating

## Goal
The server knows how much of a video someone actually watched, and the quiz
unlocks only after they have watched enough. Seeking to the end does not count.

## In scope
- A viewer identity that works for anonymous visitors
- A watch progress model recording contiguous watched time
- Heartbeats from the player, validated server side
- A per lesson watch requirement, gating the quiz
- Progress shown in the UI so the requirement is never a mystery
- Tests, including the ones that prove skipping does not work

## Out of scope
- Resume where you left off. Worth doing, but keep this feature focused.
- Per second heatmaps, drop off analytics (feature 012)
- Anti automation beyond the plausibility checks below. A determined person
  with the network tab open can still fake it, and that is acceptable.
- Applying the gate retroactively to attempts already completed

## Why this exists
Self study continuing education generally requires evidence of engagement, and
that requirement is the hardest thing in this application to retrofit, because
it touches the player, the data model, and the quiz flow at once. abacadaba is
the cheap place to learn what it costs. Build it here even though a fun
micro learning app would not strictly need it.

## The identity problem, and its answer
Watch progress needs to attach to someone, but the quiz must stay open to
anonymous visitors. On any first request without one, set a viewer_id cookie
holding a UUID: httpOnly, SameSite=Lax, Secure in production, one year. Every
watch record keys off viewer_id. When a signed in user appears, records also
carry user_id so progress follows the account across devices.

Implement this as middleware so no route has to remember it.

## Anti skip rules
The player sends a heartbeat every 10 seconds carrying its current position.
The server credits watched time only when all of these hold:
- the elapsed wall clock time since that viewer's previous heartbeat is at
  least 8 seconds, rejecting a flood of requests
- the reported position has advanced by no more than 15 seconds since the
  previous heartbeat, rejecting a seek forward
- the reported position does not exceed the lesson's duration_seconds
When the position moves backward, that is a legitimate rewind. Do not credit
time, but do accept it and reset the comparison point.

Credit the smaller of the position delta and the wall clock delta. Store the
running total in watched_seconds. A rewatched section does not earn credit
twice, because credit comes from forward movement only.

## Data model
Table watch_progress:
- id: integer primary key
- lesson_id: integer, foreign key to lessons.id, not null, indexed, CASCADE
- viewer_id: UUID, not null, indexed
- user_id: integer, foreign key to users.id, nullable, indexed, SET NULL
- watched_seconds: integer, not null, default 0
- last_position: integer, not null, default 0
- last_heartbeat_at: timezone aware UTC, nullable
- completed_at: timezone aware UTC, nullable. Stamped when the requirement is met.
- created_at, updated_at: timezone aware UTC, server defaults
- Unique constraint on (lesson_id, viewer_id)

Add to lessons:
- required_watch_ratio: numeric or float, not null, default 0.9. The fraction of
  duration_seconds that must be watched. Editable per lesson in the admin.

A lesson with a null duration_seconds cannot gate, since there is nothing to
measure against. Treat it as ungated and surface that in the admin as a warning.

## Backend tasks
1. app/models/watch_progress.py, plus required_watch_ratio on Lesson. Migration,
   inspected before applying, upgrade and downgrade verified.
2. app/middleware/viewer.py: reads or sets the viewer_id cookie and attaches it
   to request.state. Registered in main.py.
3. app/services/watch.py:
   - record_heartbeat(db, lesson, viewer_id, user_id, position) applying every
     rule above and returning the updated progress
   - get_progress(db, lesson, viewer_id) returning watched_seconds, the required
     seconds, a ratio, and whether the requirement is met
   - is_unlocked(db, lesson, viewer_id) as the single function the quiz gate
     calls. Nothing else decides this.
   Use a single upsert or a row level lock so two concurrent heartbeats cannot
   both credit the same interval.
4. app/routers/watch.py:
   - POST /lessons/{slug}/watch with a position, returning current progress
   - GET /lessons/{slug}/watch returning current progress
   Both work anonymously. Rate limit the POST to roughly one per five seconds
   per viewer and return 429 beyond that.
5. Gate the quiz: POST /lessons/{slug}/attempts returns 403 with a clear detail
   when is_unlocked is false, saying how much of the video remains. The quiz
   delivery endpoint stays open so the questions can be previewed, or gate it
   too if that feels wrong once you see it. Pick one and note the choice.
   Admins bypass the gate, so authoring can be tested without watching.
6. tests/test_watch.py:
   - a heartbeat sequence at a realistic pace accumulates watched_seconds
   - SKIP TEST: a heartbeat jumping from position 5 to position 300 credits
     nothing. Comment this as the guard the feature exists for.
   - FLOOD TEST: ten heartbeats within one second credit at most one interval
   - rewinding does not credit time and does not corrupt the total
   - starting an attempt below the threshold returns 403
   - starting an attempt at or above the threshold succeeds
   - an admin can start an attempt without watching
   - a lesson with a null duration is ungated
   - progress for a signed in user is visible from a different viewer cookie

## Frontend tasks
1. src/api/watch.js: sendHeartbeat(slug, position), getWatchProgress(slug).
2. src/components/VideoPlayer/: on timeupdate, throttle to one heartbeat every
   10 seconds and only while the video is actually playing. Stop on pause, and
   flush a final heartbeat on ended and on page hide via visibilitychange, so
   closing the tab does not lose the last stretch.
3. Show progress under the player as a bar with plain text, for example
   "3:10 of 4:41 watched". Do not make the requirement a guessing game.
4. src/pages/LessonDetail/: the Take the quiz button is disabled until unlocked,
   with text saying what is left. It enables live when the threshold is crossed,
   without a refresh.
5. Handle the 403 from attempt creation gracefully if someone reaches the quiz
   URL directly: explain and link back to the lesson.
6. Admin editor: expose required_watch_ratio as a percentage input, and show a
   warning when duration_seconds is missing since gating will not apply.

## Acceptance criteria
- watching a video normally accumulates progress and the bar advances
- dragging the scrubber to the end does not unlock the quiz
- the quiz button is disabled with a clear explanation until the threshold is met
- crossing the threshold enables the button without a page refresh
- closing the tab mid video and returning shows the progress that was already
  earned
- a signed in user's progress follows them to a different browser
- an anonymous viewer's progress survives a refresh via the cookie
- POSTing the attempt endpoint directly without watching returns 403
- an admin can bypass the gate
- pytest passes, including the skip and flood tests

## When done
Append an entry to CHANGELOG.md and stop.
