# Current Feature

## Feature 020c, Authoring hotfix

## Goal
Four defects found by authoring a course after 020a and 020b shipped. One of
them destroys work every time it fires. One of them is a fix that 020a claimed
to make and did not. Fix all four and nothing else.

## In scope
- Course description not persisting
- Quiz question numbering, reopened from 020a
- Add-question control disorients the author
- Publish checklist shows a green check for a rule with nothing to check

## Out of scope
- Background or resumable video upload. Real, acknowledged three times now,
  deliberately queued as its own feature. Do not start it here.
- Collapsing single-lesson courses onto one page. That is the next feature and
  it is larger than this one.
- Bulk question import. Certificate design (feature 024). Any schema change.
- Any further layout work beyond the specific items below. This is a hotfix.

## Bug 1, the course description does not persist

**This is the priority. It silently discards content the author typed.**

**Reproduce:** open a course editor, type into Course description, save, reload
or navigate back to the course.
**Observed:** the field is empty. Retyping and saving again does not help.
**Expected:** it persists like every other field on the page.

Do not start from the hypothesis below — reproduce it first and confirm the
mechanism. But the evidence points somewhere specific, so start looking there.

The smoking gun is that the publish checklist read `○ Description` at the same
moment the textarea on screen was full of text. The checklist renders from
server state and the textarea renders from client state. They disagreed, which
means the value was never persisted, rather than persisted and then lost on
read. That rules out a load or hydration problem and points at the save.

020b replaced per-section saves with one batched save for the page. The most
likely mechanism is that the batch is assembled from fields that registered a
change through the new dirty-tracking path, and the description textarea is not
wired into it — so it is typed, looks correct, and is dropped when the payload
is built.

Whatever the mechanism turns out to be, check every field on both editors for
the same defect rather than fixing only the one that was reported. If the
description fell out of the batch, ask what else did. Prerequisites, advance
preparation, retake cooldown, and max attempts are all on that form and none of
them were exercised in the failing session.

Add a test that saves every field on the course editor in one batch and asserts
each one round-trips. A test for the description alone would not have caught the
class of bug and will not catch the next one.

## Bug 2, quiz numbering, reopened

020a specified this and the changelog should say it shipped. It did not.

**Observed:** the assessment shows `Question 1 of 5` in the progress line above a
card headed `5. Why can some volcanic eruptions be highly explosive?`

The diagnosis in 020a stands and is unchanged: the shuffle is correct and
deliberate, and the card is printing the authored `position` instead of the
index within the shuffled run.

Find out why the fix did not take before making it again. Either it was applied
to a component the quiz page does not use, or the number is composed somewhere
shared — a prompt string built as `${position}. ${text}` upstream of the card
would survive a change to the card itself.

Then add a regression test that fails against the current build. 020a passed its
acceptance criteria without this working, which means it was checked by reading
the code rather than by running the quiz.

## Bug 3, questions are added at the top and appear at the bottom

020b moved the add-question control above the list, outside the question cards,
to stop it being mistaken for the add-choice input. That worked — the mis-entry
that produced a question called "Lava" has not recurred.

It also created a new problem: the author types a question at the top of the
section, the question is appended to the end of the list, and nothing indicates
anything happened. With four questions authored, the new one is off screen.

**Keep the control where it is.** Moving it back below the last question would
put it directly under that question's `New choice text` input again, which is
the exact adjacency that caused the original defect. Do not undo 020b to fix
this.

Instead, make the result visible:
- After adding, scroll the new question into view and focus its prompt textarea.
- The section heading count updates, which it already does.

If scroll-into-view proves awkward inside the page's scroll container, say so
rather than working around it by relocating the control.

## Bug 4, a green check for a rule with nothing to check

With `Lessons (0)`, the publish checklist renders `✓ Every lesson has a video,
at least one question, and each question has exactly one correct choice` in
green, directly under `○ At least one lesson`.

The rule is vacuously true — there are no lessons, so no lesson fails it. But a
green check tells the author that part is done.

A rule whose subject set is empty renders neutral, not satisfied. Apply that as
a rule rather than special-casing this one line: any per-lesson check with zero
lessons is `○`, not `✓`.

## Compliance
Bug 1 destroys the course description, which is the pre-enrollment disclosure
mapped to 8.01.1 in COMPLIANCE.md. That mapping currently claims a satisfied
requirement that the application does not actually deliver.

Check whether the existing 8.01.1 row needs its Gap column updated to record
that this was broken between 020b and 020c, or whether the row is fine now that
it works. Say which you concluded. Do not add a new row for a fix that restores
intended behaviour.

The other three bugs are expected to map to no locator. Confirm rather than
assume, and say so explicitly.

## Backend tasks
1. Only if bug 1 turns out to be server-side. Diagnose first.
2. `tests/`: a test that saves every field on the course editor's details form in
   one request and asserts each round-trips.
3. `tests/`: a regression test for quiz question numbering that fails against the
   current build.

## Frontend tasks
1. Fix whatever drops the description from the batched save; audit every other
   field on both editors for the same defect.
2. Number quiz questions by index within the served order, everywhere the number
   is composed.
3. Scroll to and focus a newly added question.
4. Per-lesson publish checks render neutral when there are no lessons.

## A thing to check rather than assume
020a's acceptance criteria included "the assessment's card number matches its
progress counter on every question" and that criterion was reported met while
the bug was live. Before closing this feature, take the quiz in a browser as a
non-admin user and look at the screen. Do not report bug 2 fixed on the strength
of a passing test alone.

## Acceptance criteria
- a course description survives save, reload, and navigating away and back
- every other field on the course details form survives the same round trip
- the assessment's card number matches its progress counter, verified by taking
  the quiz in a browser, not only by test
- retaking still produces a different order than the first attempt
- adding a question brings it into view with its prompt focused
- adding a question is still not confusable with adding a choice
- a course with no lessons shows no green checks for per-lesson rules
- `npm run lint` passes
- pytest passes

## When done
Append an entry to CHANGELOG.md, including a note that bug 2 was reported fixed
in 020a and was not. Update COMPLIANCE.md only per the Compliance section above,
and say explicitly what you concluded. Then stop.
