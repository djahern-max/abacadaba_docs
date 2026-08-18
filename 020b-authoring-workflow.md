# Current Feature

## Feature 020b, Authoring workflow

## Goal
Authoring a course stops requiring knowledge of the data model. One save model
instead of four, fields that say what they are for, a control that cannot be
mistaken for a different control, and a visible next step at every point.

## In scope
- One save model across both editors
- Labelling the course/lesson field pairs that currently look duplicated
- Separating the add-question control from the add-choice control
- A next step at the bottom of every editor, not only a link at the top
- Orienting a new author on an empty course

## Out of scope
- The three defects in 020a. Run that first; this feature assumes it shipped.
- Background or resumable video upload. Real, scheduled, not here — it changes
  the upload transport and this feature only changes the page around it.
- Bulk question import from a template. Also real, also not here, and it wants
  the feature 023 review/assessment split to exist first so the template does
  not need versioning three weeks in.
- Certificate content or design. Feature 024.
- Any schema change. This is entirely presentation and client state.
- Changing what publish validation requires. 020a adjusts the rules; this
  feature only changes how they are surfaced.

## Where this came from
One authoring session, start to finish, by the person who commissioned every
feature in the sequence. The verdict was "I am building the thing and I don't
even understand the way it is supposed to work." Every item below is something
that happened in that session, not something anticipated.

Features 010 through 020 were each specified and built in isolation, and each is
internally coherent. The path across them is not. That is the defect.

## Part 1, one save model

Right now the lesson and course editors use four different rules on one page:

- Details: one Save button for the whole section
- Learning objectives: a Save button per row
- Questions: a Save per question, plus a Save per choice
- Video and thumbnail: no save at all, upload begins on file selection

There is nothing to learn here, because it is not a rule. An author who has
internalised "Save is per row" then loses a details edit, and one who has
internalised "it saves itself" loses a question prompt — which is exactly what
020a is fixing the consequences of.

**Decision: keep explicit save, make it one per editor page.**

Explicit save is right and should not become autosave. A published course is a
document participants are working through and that a sponsor has to be able to
produce on audit; edits landing on it keystroke by keystroke is the wrong
behaviour for that object regardless of how convenient it feels in the editor.
The problem was never that saving is explicit. It is that "explicit" was
implemented at four different granularities.

So:
- Edits to details, objectives, questions, and choices are held in client state
  and committed by one Save action for the page.
- A sticky bar at the bottom of the editor is the only Save control. It shows a
  count of unsaved changes and is inert when there are none.
- Every per-row Save button is removed. Row-level Delete, Move up, and Move down
  stay immediate — they are actions, not field edits, and batching a delete
  behind a save is worse than not batching it.
- Uploads stay immediate. They are file transfers, not form fields, and holding
  a 200 MB file in client state until a Save click would be a lie about what is
  happening. Say so in the section: the upload section explicitly states that
  video and thumbnail save on selection, unlike everything else on the page.

Keep 020a's dirty tracking as the mechanism. This feature changes what the
author does with it, not how it is computed.

If batching question and choice edits turns out to need a new bulk endpoint,
stop and say so before building one — that is a backend change this feature
claims not to need, and it is worth a decision rather than a quiet migration.

## Part 2, the duplicated-looking fields

A course has a thumbnail and a description. Each lesson inside it also has a
thumbnail and a description. The author's reaction on hitting the second pair
was "What happens with the first thumbnail??"

Nothing happens to it. All four are used and none overwrite each other. But
nothing on either page says so, and the fields are labelled identically.

Label each with where it appears:
- Course thumbnail: shown on the course card in the catalog.
- Course description: shown on the public course page before enrollment. This is
  the pre-enrollment disclosure required by 8.01.1, so it is the one that
  matters most and should read as the more consequential of the two.
- Lesson thumbnail: the poster frame shown before this segment's video plays.
- Lesson description: a short blurb for this segment in the course's lesson list.

Helper text under the field, in the same style as the existing "Blank means
unlimited attempts." lines. Not a tooltip.

## Part 3, the add-question control

`New choice text` / `Add choice` sits directly above `New question prompt` /
`Add question`. Same width, same shape, a few pixels apart. The author put an
answer into the question box three times in three attempts.

That is a control problem, not an attention problem. Fix it structurally:

- Choices are visually nested inside their question — indented, with a rule or
  background distinguishing the question's block from the page.
- The add-question control moves out of the flow of the last question's choices.
  Put it in the Questions section header, or make it a full-width button clearly
  outside every question block. It must not be the next thing under a choice
  input.
- The two inputs get different shapes. A question prompt is a textarea, a choice
  is a single-line input; make that visible rather than rendering both as
  same-sized boxes.

Same treatment for objectives if the equivalent adjacency exists there.

## Part 4, the next step

Finishing a lesson leaves the author at the bottom of the page with a Delete
lesson button and nothing else. The only route onward is scrolling to the top
and finding a hyperlink inside a sentence.

- Both editors end with an explicit next action, below the content and above the
  danger zone. In a lesson: back to its course, plus what is still outstanding on
  this lesson. In a course: publish, or what is blocking it.
- The course publish checklist's per-lesson failures become links to that
  lesson's editor. "Lesson 'X' must have a video" should be clickable.
- The danger zone stays last and stays visually separate. Delete is not a next
  step.

## Part 5, the empty course

A new course opens on an empty editor with eight fields and `Lessons (0)`, and
does not say that lessons are where video and questions live. The author went
looking for the question editor on the course page.

When a course has no lessons, the Lessons panel says what a lesson is: one video
segment plus its questions, and a course is one or more of them in order. One or
two sentences in the panel, replaced by the list once a lesson exists. Not a
modal, not a dismissible tour.

## Compliance
This feature changes how existing disclosures are labelled in the admin tool. It
does not change what is disclosed to a participant, so the expectation is that
COMPLIANCE.md gains no row — the 8.01.1 and 3.02.1 mappings from feature 020 are
unchanged and still accurate. Confirm that against
`docs/2026-Statement-on-Standards-for-CPE-Programs.pdf` and say so explicitly
rather than leaving it unstated.

If Part 2's relabelling reveals that a field mapped in 020 is not actually
rendered where COMPLIANCE.md claims, that is a real gap and it goes in the Gap
column. Check it while you are in there.

## Backend tasks
None expected. If a task appears to need one, stop and check whether existing
endpoints can carry it before adding anything.

## Frontend tasks
1. One sticky Save per editor page; per-row Save buttons removed; uploads
   exempted and labelled as such.
2. Helper text on both thumbnail and both description fields.
3. Questions restructured: choices nested, add-question moved out of the choice
   flow, textarea vs input distinction.
4. Next-step block at the bottom of both editors; publish checklist lesson
   failures link to their lesson.
5. Empty-state text in the Lessons panel.

## A thing to check rather than assume
020a wires dirty tracking through the questions and objectives editors. This
feature depends on that being in place and correct. Confirm it before starting;
if 020a has not shipped, stop rather than building a second dirty-tracking path
that will have to be reconciled.

## Acceptance criteria
- there is exactly one Save control per editor page, and it reports how many
  unsaved changes it will commit
- editing a question, an objective, and a details field, then saving once,
  persists all three
- the upload sections state that they save on selection
- each thumbnail and description field says where it appears
- adding a choice and adding a question are visibly different actions in
  different places, and choices read as belonging to their question
- the bottom of a lesson editor offers a route back to its course and says what
  is outstanding
- a publish checklist entry naming a lesson navigates to that lesson
- a course with no lessons explains what a lesson is
- authoring a complete course start to finish requires no scrolling back to the
  top to find the way forward, and no prior knowledge of the data model
- `npm run lint` passes
- pytest passes, unchanged

## When done
Append an entry to CHANGELOG.md. Append to COMPLIANCE.md only if a locator
genuinely applies, and say explicitly in your summary if none does. Then stop.
