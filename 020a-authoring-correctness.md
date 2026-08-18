# Current Feature

## Feature 020a, Authoring correctness fixes

## Numbering convention
A letter suffix means corrective work against the base feature's surface area:
`020a` fixes and `020b` redesigns what `020` and its predecessors shipped. The
numbered sequence (021 onward) stays reserved for new capability. Record this in
CLAUDE.md as part of this feature so future sessions follow it.

## Goal
Three defects found by authoring a complete course end to end for the first
time. Each one is reproducible and each one is small. Nothing in this feature
changes layout, wording, or workflow — that is 020b.

## In scope
- Quiz question numbering
- Unsaved edits in the questions, choices, and objectives editors
- Publish validation requiring choices on a question

## Out of scope
- Anything about how the editor is laid out or worded. 020b.
- Background/resumable video upload. Scheduled separately.
- Bulk question import. Scheduled separately.
- Certificate content and design. Feature 024 already covers this.
- Any schema change. If a task here appears to need a migration, stop and say so
  rather than adding one.

## Bug 1, the quiz numbers questions by authored position

**Reproduce:** author a five-question lesson, publish, take the assessment.
**Observed:** the progress line reads "Question 1 of 5" while the question card
reads "4. Lava". Advancing goes 4, then 1, then 3.
**Expected:** the card number matches the progress counter.

This is a display bug only. Feature 012 shuffles question order per attempt from
`shuffle_seed`, deliberately, so a retake does not replay the same order. The
order being served is correct. The quiz page is printing the question's authored
`position` field instead of its index within the shuffled run, so two numbers
describing the same thing disagree on one screen.

Fix in the frontend. Render the index within the served list, one-based. Do not
change the shuffle, the seed, or what the API returns — and specifically do not
"fix" this by serving questions in authored order, which would silently undo
feature 012.

Check the result page and any per-question review view for the same mistake
before assuming the quiz page is the only place it appears.

## Bug 2, an unsaved question prompt publishes silently

**Reproduce:** create a question, type its prompt, add its choices, mark one
correct, but do not click Save on the prompt itself. Publish the course.
**Observed:** publish succeeds. The public assessment shows the stale prompt —
in the real case, an answer string that had been typed into the question box by
mistake and then corrected but never saved.
**Expected:** either the edit is saved, or publish refuses and says which field
is unsaved.

Feature 017 built dirty tracking, a disabled Publish with a reason, and a
navigate-away warning. All of it was wired to `DetailsForm` only. `QuestionsEditor`,
the per-choice inputs, and `ObjectivesPanel` were never included, so the editor
tracks unsaved state in one section and is blind to it in three others.

Extend the existing mechanism rather than building a second one:
- Every editable text input in the lesson and course editors reports dirty state
  up to the page, the same way `DetailsForm` does via `onDirtyChange`.
- `hasUnsavedWork` on both editors includes all of them, so the existing
  `beforeunload` handler and the back-link confirm already cover them.
- Publish is disabled while anything is dirty, with the existing "Save your
  details first" reason generalised to name what is unsaved.

The per-row Save buttons stay exactly as they are in this feature. Whether the
save model should change at all is 020b's question, and answering it here would
put a layout decision inside a bug fix.

## Bug 3, publish accepts a question with no choices

Not observed in the failing run — the question that published with a bad prompt
did have its four choices by then. This is a gap found by reading
`validate_for_publish`, not a reproduced defect, and it is in scope only because
it is two lines next to work already being done.

Add to the course publish validation: every question must have at least two
choices. The existing "exactly one correct choice" rule does not catch a
question with zero choices in a useful way — the message a person needs is
"this question has no answers", not "no correct choice is set".

Keep it in the same flat list of every failure at once that feature 010
established. Do not short-circuit on the first failure.

## Compliance
Check whether any of this maps to a locator in
`docs/2026-Statement-on-Standards-for-CPE-Programs.pdf` before writing to
COMPLIANCE.md. The expectation is that it does not — this is correctness in an
authoring tool, not a disclosure or documentation requirement — but confirm it
rather than assuming, and say so explicitly in the summary if you conclude there
is no row to write. Do not invent a locator to have something to append.

## Backend tasks
1. `app/services/admin_content.py`: add the minimum-two-choices rule to
   `validate_for_publish`, message naming the offending question.
2. `tests/test_admin_content.py`: publishing a course whose question has no
   choices returns 422 and the message identifies the question.

## Frontend tasks
1. Quiz page: number questions by their index in the served order.
2. `QuestionsEditor`, the choice inputs, and `ObjectivesPanel`: report dirty
   state to their parent editor.
3. `AdminLessonEditor` and `AdminCourseEditor`: fold that into `hasUnsavedWork`
   and into the Publish disabled reason.

## A thing to check rather than assume
Feature 012's leak test asserts `is_correct` appears in no public payload.
Changing how the quiz page reads its question list should not touch that, but
run the leak test and confirm rather than assuming the change is presentational.

## Acceptance criteria
- the assessment's card number matches its progress counter on every question,
  across two attempts with different shuffles
- retaking still produces a different order than the first attempt
- editing a question prompt without saving disables Publish and names it
- navigating away with an unsaved question prompt warns
- saving that prompt then publishing produces the corrected text on the public
  assessment
- publishing a course whose question has no choices is refused, with a message
  identifying the question
- the leak test passes
- `npm run lint` passes
- pytest passes

## When done
Append an entry to CHANGELOG.md. Append to COMPLIANCE.md only if a locator
genuinely applies, and say explicitly in your summary if none does. Then stop.
