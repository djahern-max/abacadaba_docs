# Current Feature

## Feature 019a, The single-lesson course

## Numbering
This is `019a`, not `020e`. It amends the hierarchy feature 019 introduced —
what a course is made of and how many pages that costs — rather than the save
model 020 and its suffixes have been working on. 020d filed it under this
number; keep it there.

## Goal
A course with one lesson is authored on one page and taken on one page. The
two-level model stays exactly as 019 built it in the database. It stops being
something the author or the participant has to walk through.

## In scope
- Creating a course creates its first lesson
- A collapsed admin editor for courses with exactly one lesson
- Expanding to the two-level editor when a second lesson is added, and
  collapsing back when it is removed
- The public course page playing its only segment inline
- Refusing to delete the last remaining lesson of a course

## Out of scope
- Any schema change. Specifically: no `is_single_lesson` column, no mode flag,
  no denormalised copy of lesson fields onto the course. See "One rule, derived"
  below — this is the single most important constraint in the feature.
- Multi-lesson authoring. A course with two or more lessons keeps the editor it
  has today, unchanged.
- Background or resumable video upload. This is the fourth feature file to say
  so. It is real, it is queued, it changes the upload transport, and it is still
  not this.
- Bulk question import.
- The development and review chain. Feature 021.
- Credit calculation. Feature 022.
- Certificate content or design. Feature 024.
- Changing what `validate_for_publish` requires — with one exception, flagged in
  Backend task 3, where a rule may turn out to demand a field the collapsed page
  no longer shows. That is a genuine conflict, not a rule change for its own
  sake.

## Where this came from
020c named it: "Collapsing single-lesson courses onto one page. That is the next
feature and it is larger than this one."

Underneath that, the product's premise. abacadaba is a micro-learning platform.
The overwhelmingly common shape is one video, one set of questions, one
certificate. Feature 019 was right that credit attaches to a program rather than
to a video, and it was right to make that structural before anything depended on
it. The cost, unbudgeted at the time, is that the minimum path to publishing one
video is now two pages, two editors, two Save buttons, and an understanding of
the difference between a course and a lesson — which is precisely the knowledge
020b set out to stop requiring, and the one instance of it 020b could not fix by
relabelling.

## One rule, derived
**A course renders collapsed when it has exactly one lesson.** That is the whole
rule. It is computed from the lesson count on every render, in the admin and on
the public side, and it is never stored.

Do not add a flag. A flag means two sources of truth for one fact, a course that
can get stuck in expanded mode with one lesson in it, a migration to fix the
ones that do, and a decision about what the flag means when a lesson is deleted.
The derived rule answers all of that by not raising the question. If a task in
this feature seems to need a stored mode, stop and say so rather than adding
one.

The consequence is that the transition runs both ways and is not special: adding
a second lesson expands, deleting back down to one collapses. Nothing is lost
when a course collapses — hidden lesson fields keep their values in the
database, they are simply not rendered.

## Part 1, creation makes a lesson
`POST` course creation creates lesson 1 in the same transaction, at position 1,
with its title set to the course title.

Do this on the server, not as two calls from the frontend. A failed second call
leaves a course with zero lessons, which is a state nothing in the product can
render and nothing can publish. Making it atomic means that state never exists.

Multi-lesson courses get their first lesson this way too, which is correct — the
author was going to create one anyway.

The lesson title is set once and never synchronised afterwards. Syncing means
deciding what happens when the two diverge, and the value is invisible while the
course is collapsed. On expansion the author sees it in the Lessons panel and
can rename it there.

## Part 2, the collapsed editor
On a course with one lesson, one page shows:

- Course title, slug, description, thumbnail
- Learning objectives, program level, field of study, prerequisites, advance
  preparation
- Retake cooldown, max attempts
- The video and its auto-filled duration
- The questions editor
- The publish panel

And hides, without deleting:

- Lesson title, lesson description, lesson thumbnail, lesson position

This resolves the field pairs 020b Part 2 had to label its way around. On a
collapsed course there is only one description and one thumbnail, so the helper
text explaining which is which has nothing to disambiguate and should not
render. Check that the labels 020b added are conditional rather than hardcoded
into the shared components.

There is no Lessons panel and no link to a lesson editor. The lesson editor
route still exists and still works; nothing on a collapsed course links to it.

## Part 3, the transition
The control is a single action at the bottom of the video and questions region,
and **its label states the consequence before the click**:

> Add a second segment — this course will split into a course page and a page
> for each segment.

That is the pattern 020d established. The author is told what will happen while
they can still choose not to, rather than being shown an explanation after their
page has been replaced.

On click: create lesson 2 and navigate to its editor, because filling it is the
reason the control was pressed. The course editor behind it is now the
two-level version, with a Lessons panel listing both.

Collapsing back is the same rule in reverse and needs no control of its own.
Deleting lesson 2 leaves one lesson and the course renders collapsed on the next
load.

One edge worth writing down rather than discovering: an author who expands, puts
content into lesson 2, then deletes lesson 1 ends up with a collapsed course
whose visible content is lesson 2's, and whose lesson title, description, and
thumbnail are the ones they set on lesson 2 and can no longer see. That is
correct behaviour under the derived rule. No data is lost.

## Part 4, the public page
A participant on a one-lesson course sees one page: the disclosure block, then
the player, then the assessment button with its gate.

Order matters and is not negotiable. The objectives, program level, field of
study, prerequisites, and advance preparation that feature 020 put on this page
stay **above** the player. 8.01.1 asks for significant features to be disclosed
in advance, and 020's changelog records that they were deliberately placed above
the assessment button so they are pre-enrollment rather than something found
after starting. Putting the player above them would move a decision the
participant is supposed to make beforehand to after they have started watching.

`/courses/:slug/lessons/:n` on a one-lesson course redirects to the course page
rather than 404ing. Cheap, and certificates, bookmarks, or old links may point
at it.

**The watch gate does not change.** Feature 011's gate, its progress reporting,
and the assessment button's locked state all behave exactly as they do now; the
only difference is where the player element renders. The specific thing to
verify is that progress still posts from the inline player, because if it does
not, the gate never opens and the course becomes uncompletable — a failure that
looks like a content problem rather than a layout one.

## The leak test
The public course payload is fetchable signed out. The video URL is not, and
must not become so.

If the course payload grows the fields the inline player needs, the playable
video URL is not one of them for an anonymous request. Serve it exactly as
gated as the lesson segment endpoint serves it today. Feature 020's leak test
exists for this; extend it to cover the new payload rather than assuming the
existing assertion still covers the right surface.

## Backend tasks
1. Course creation creates lesson 1 in the same transaction. Title from the
   course title, position 1.
2. `delete_lesson` refuses when the lesson is its course's only one. 409, with a
   message saying to delete the course instead. This mirrors the existing 409 on
   deleting a course with completed attempts.
3. **Check before building:** read `validate_for_publish` and list every rule it
   applies to a lesson. If any of them require a field the collapsed editor
   hides — lesson description, lesson thumbnail — then a course authored on the
   collapsed page cannot be published from it, which is worse than the problem
   this feature is fixing. Resolve it explicitly: either the field stays visible
   on the collapsed page, or the rule is scoped to courses with more than one
   lesson. Say which you chose and why in the changelog. Do not discover this
   during acceptance testing.
4. Whatever the public course payload needs for the inline player, subject to
   the leak test above.
5. Tests:
   - creating a course yields exactly one lesson, at position 1
   - deleting the only lesson of a course is refused, and the course and lesson
     both still exist afterwards
   - deleting one of two lessons succeeds
   - the anonymous course payload contains the 020 disclosures and no video URL
   - a signed-in participant can play, and progress recorded from the course
     page satisfies the gate
   - the existing leak test still passes

## Frontend tasks
1. `AdminCourseEditor` branches on lesson count: collapsed layout at exactly
   one, existing layout otherwise. One branch, at the top, not scattered
   conditionals through five components.
2. The add-second-segment control, with the consequence in its label.
3. Conditional field-pair helper text, per Part 2.
4. `CourseDetail` renders the player inline when the course has one lesson.
5. Redirect the segment route for one-lesson courses.
6. Publish checklist entries for the single lesson drop their `Lesson 'X'`
   prefix. On a collapsed course there is no second lesson to disambiguate from,
   and the prefix names something the author cannot see. The 020b behaviour
   where the entry links to that lesson's editor should also not fire — there is
   nowhere else to go.

## A thing to check rather than assume
This feature sits on top of 020d's save model. The collapsed page holds course
fields and lesson fields in one form, committed by one Save. Confirm that the
batched save from 020b/020d can already commit to both a course and its lesson
in one action, or that it can be extended to without a second save path. If
020d has not shipped, stop — building the collapsed editor against a save model
that is about to be replaced means doing it twice.

## Acceptance criteria
- creating a course lands the author on one page containing every field needed
  to publish, with no link to a second editor
- that page publishes a complete course without the author ever encountering the
  word "lesson" as something they have to act on
- adding a second segment states, before the click, that the page will split
- after adding a second segment, the course editor shows a Lessons panel with
  both segments and the first one carries the title it was created with
- deleting the second segment returns the course to the one-page editor with all
  of its content intact
- deleting the only segment is refused, with a message that says what to do
  instead
- a signed-out visitor to a one-lesson course sees objectives, program level,
  field of study, prerequisites, and advance preparation above the player
- a signed-in participant plays the video on the course page, and the assessment
  button unlocks on the same page without a navigation
- `/courses/:slug/lessons/1` on a one-lesson course redirects rather than 404s
- a two-lesson course behaves exactly as it does today, on both sides
- `npm run lint` passes
- pytest passes, with the new tests added

## When done
Append an entry to CHANGELOG.md, including the answer to Backend task 3.

Then check COMPLIANCE.md. The expectation is that it gains no row: this feature
changes where disclosures render, not what is disclosed, and 020's 8.01.1 and
3.02.1 mappings should still be accurate. Confirm that against
`docs/2026-Statement-on-Standards-for-CPE-Programs.pdf` and say so explicitly
rather than leaving it unstated — the mapping now points at a different page
template than it did when it was written, so it is worth re-reading rather than
re-assuming.

If the disclosure block ended up below the player, that is a real gap and it
goes in the Gap column rather than being quietly accepted.

Then stop.
