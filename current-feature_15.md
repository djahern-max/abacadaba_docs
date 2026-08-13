# Current Feature

## Feature 015, Watch progress correctness

## Goal
Watch progress belongs to one person. Two users sharing a browser stop
inheriting each other's history, and the progress readout stops showing numbers
that cannot be true.

## In scope
- Per-user watch progress, with a migration
- Rotating viewer_id on sign out
- A progress bar that moves with playback rather than in 10 second steps
- Fixing the "5:09 of 4:41 watched" readout
- Tests for the leak this feature exists to close

## Out of scope
- Requiring sign in to take a quiz. That is feature 016 and it overlaps, but the
  leak must be fixed independently — anonymous viewers will still have watch
  progress afterwards.
- Resuming playback where you left off
- Per second heatmaps, still deferred from feature 011

## The bug, stated precisely
`app/services/watch.py` looks up progress by `viewer_id` OR `user_id`.
`viewer_id` is a one year cookie set per browser by `ViewerIdentityMiddleware`,
not per account. So:

1. User A signs in, watches the lesson. A row is written carrying A's
   `viewer_id` and A's `user_id`.
2. A signs out. The session cookie clears; the `viewer_id` cookie does not.
3. User B signs in on the same browser. The OR lookup matches A's row on
   `viewer_id`, and B sees A's progress.

The OR was written so a signed-in user's progress follows them to a new device.
That is a good goal, and this feature keeps it — but the two halves of the OR
must not both apply at once.

## The fix
Resolve progress by identity, not by union:

- Signed in: match on `user_id` only.
- Anonymous: match on `viewer_id` only, and only for rows with a NULL `user_id`.

A signed-in user still finds their progress on a new device, because `user_id`
travels with the account. Another user on the same browser no longer matches,
because the `viewer_id` half never applies to them.

Additionally, rotate `viewer_id` on sign out. Defence in depth: even if some
future query reintroduces a viewer-based match, the cookie no longer points at
the previous person's row.

## Claiming anonymous progress at sign in
Someone who watches four minutes signed out and then signs in should keep that
progress rather than starting over.

On successful sign in (all three paths: register, login, Google callback), for
each `watch_progress` row matching the current `viewer_id` with a NULL
`user_id`, either set `user_id` to the new user, or merge into that user's
existing row for the lesson by taking the larger `watched_seconds`. Do this in
one transaction, before the redirect.

Merge rather than overwrite. Taking the maximum is right; taking the anonymous
value unconditionally would let someone lose progress.

## Data model
`watch_progress` currently has a unique constraint on `(lesson_id, viewer_id)`.
That is now wrong: one user can legitimately have rows from several browsers,
and one browser can carry rows for several users.

Replace it with two partial unique indexes:
- unique on `(lesson_id, user_id)` where `user_id IS NOT NULL`
- unique on `(lesson_id, viewer_id)` where `user_id IS NULL`

Write these as explicit `op.create_index(..., postgresql_where=...)` in the
migration. Autogenerate does not reliably produce partial indexes — inspect the
generated file and expect to write them by hand.

Before adding the new indexes, collapse any existing duplicate rows, or the
index creation fails on live data. Your local database is freshly wiped so this
will pass silently in development and bite in production.

## The two display bugs
**"5:09 of 4:41 watched."** The label compares watched seconds against the
*required* seconds, so it reads as nonsense the moment the requirement is
exceeded. Show progress against the lesson's full duration ("5:09 of 5:12"),
and once the threshold is met replace the counter with a plain "Watched" state.
The unlock threshold is a property of the gate, not a thing to measure against.

**The bar advances in 10 second jumps.** The heartbeat fires roughly every 10
seconds and the bar is driven by the server's response. Keep the heartbeat
interval — it exists to limit write volume and is rate limited at about one per
5 seconds anyway — but drive the *bar* from the player's own `timeupdate`
event, which fires several times a second. The server response remains the
source of truth and corrects the bar whenever it arrives.

Do not shorten the heartbeat interval to smooth the animation. That trades a
display concern for real database load and would trip feature 011's own 429.

## Backend tasks
1. Migration: drop the old unique constraint, add the two partial indexes.
   Verify `downgrade -1` restores the original constraint.
2. `app/services/watch.py`: replace the OR lookup with the identity-based
   resolution above. This is the security change; keep it in one function so
   there is a single place to read.
3. `app/services/watch.py`: add `claim_anonymous_progress(db, viewer_id, user)`
   implementing the merge, and call it from register, login, and the Google
   callback.
4. `app/routers/auth.py`: rotate the `viewer_id` cookie on logout.
5. `tests/test_watch.py`, add:
   - user A's progress is not visible to user B sharing a viewer_id
   - a signed-in user's progress is found from a different viewer_id
   - an anonymous viewer's progress is not visible to a signed-in user who has
     not claimed it
   - anonymous progress is claimed on sign in, taking the maximum
   - claiming twice is idempotent
   - the existing FLOOD and SKIP tests still pass

## Frontend tasks
1. `VideoPlayer`: drive the progress bar from `timeupdate`, keeping the
   heartbeat cadence unchanged. Reconcile to the server value on each response.
2. Change the readout to measure against total duration, and render a "Watched"
   state once unlocked.
3. Confirm the "Take the quiz" button still flips live at the threshold without
   a refresh, as feature 011 required.

## Acceptance criteria
- alembic upgrade head swaps the constraint for the two partial indexes,
  downgrade -1 reverses it
- user A watches a lesson, signs out, user B signs in on the same browser: B
  sees 0:00 watched
- user A signs back in on that browser and sees their own progress again
- user A signs in on a second browser and sees their progress there
- watching signed out, then signing in, preserves the watched time
- the progress bar advances smoothly during playback, not in visible steps
- the readout never shows a watched time greater than the value it is measured
  against
- once the threshold is met the readout shows a watched state rather than a
  counter
- pytest passes, including the leak test and the FLOOD/SKIP tests

## When done
Append an entry to CHANGELOG.md and stop.
