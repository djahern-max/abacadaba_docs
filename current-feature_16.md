# Current Feature

## Feature 016, Require sign in to take a quiz

## Goal
A signed out visitor can browse lessons and watch videos, but is asked to sign
in before starting a quiz, rather than taking one anonymously.

## In scope
- Attempts require an authenticated user
- A clear sign in prompt in place of the quiz link
- Returning the visitor to the lesson after they sign in
- Updating the tests that assume anonymous attempts

## Out of scope
- Removing the anonymous code paths from features 006 and 007. Historical
  anonymous attempts and their certificates stay valid and verifiable.
- Requiring sign in to watch a video, or to browse
- Any change to the pass threshold or the watch gate

## Read this before starting
This reverses a decision, so be deliberate about it. Features 005 through 007
went out of their way to let anonymous people take quizzes and claim
certificates with a typed name, and feature 008 kept `attempts.user_id`
nullable specifically to preserve that.

This feature closes the door on *new* anonymous attempts. It does not delete the
history. `attempts.user_id` stays nullable, existing rows keep working, and the
verification page keeps its self-reported-name wording for certificates issued
before this shipped.

Expect the bulk of the work to be test updates, not application code. A large
number of existing tests start an attempt without signing in.

## What changes
`POST /lessons/{slug}/attempts` requires a user. Signed out, it returns 401 with
a readable detail rather than creating an anonymous attempt.

Keep the existing 403 watch-gate check and the 429 retake-policy check. Order
matters: authenticate first, then gate, then policy. A signed out visitor should
be told to sign in, not told how much video is left.

The downstream endpoints — answering within an attempt, reading the result,
claiming a certificate — need no new guard. An attempt can no longer exist
without an owner, so they inherit the requirement.

## Backend tasks
1. `app/routers/attempts.py`: add `require_user` to the attempt creation route.
2. `app/services/attempts.py`: `start_attempt` no longer accepts a null user.
   Remove the anonymous branch of the retake-policy keying — the `viewer_id`
   fallback added in feature 012 becomes dead code for new attempts. Delete it
   rather than leaving it unreachable, and drop the note about a cleared cookie
   resetting an anonymous count, which no longer applies.
3. Tests: update every test that starts an attempt to sign in first. Add:
   - starting an attempt while signed out returns 401
   - the 401 fires before the watch gate, for a lesson the visitor has not
     watched
   - a signed-in user's flow is unchanged

## Frontend tasks
1. `LessonDetail`: when signed out, replace the quiz button and the watch-gate
   explanation with a sign in prompt linking to `/login`. Pass the lesson path
   as `state.from` so Login returns the visitor here — `Login.jsx` already reads
   `location.state?.from`.
2. `Quiz` page: reaching the quiz URL directly while signed out should redirect
   to `/login` with the same `from` state, not render an error. Signed out is a
   fixable condition, unlike the 403 gate, so redirect rather than explain.
3. `Result` page: the anonymous name-entry form from feature 007 is now
   unreachable for new attempts. Leave the component in place for old links but
   do not spend time on it.

## A thing to check rather than assume
Feature 013 made `AuthContext` resolve on mount. Confirm the lesson page does
not flash the sign-in prompt before auth resolves — a signed-in user should
never see "sign in to take the quiz" for a frame. Render a neutral state while
auth is loading.

## Acceptance criteria
- signed out, the lesson page shows a sign in prompt instead of the quiz button
- clicking it lands on /login and, after signing in, returns to the lesson
- navigating directly to a quiz URL signed out redirects to /login
- `curl -X POST` on the attempts endpoint with no cookie returns 401
- a signed-in user's quiz flow is unchanged end to end, including the watch gate
  and the certificate
- an existing certificate issued to an anonymous attempt still verifies
- no signed-in user ever sees the sign-in prompt flash on load
- pytest passes

## When done
Append an entry to CHANGELOG.md and stop.
