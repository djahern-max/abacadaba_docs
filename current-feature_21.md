# Current Feature

## Feature 021, Development and review chain

## Goal
Every published course names the subject matter expert who developed it, a
different one who reviewed it, and the date the review happened. A course whose
content has changed since its review cannot be published until it is reviewed
again. The review date appears on the course documentation, where 4.01 requires
it.

## In scope
- A `subject_matter_experts` table, deliberately not tied to `users`
- Developer, reviewer, review date, and review notes on a course
- A `sources` table for the references a course was built from
- `content_updated_at`, and publish refusing a review that predates it
- The licensed-CPA participation rule for accounting, auditing, and tax fields
- The most recent publication, revision, or review date, disclosed
- The review cycle: annual for frequently-changing subjects, biennial otherwise

## Out of scope
- The overdue-review dashboard. Feature 026. This feature stores the date and
  the cycle; 026 reports on what has aged past it. Storing the inputs without
  the report is the correct half to build first.
- Instructor qualifications, 4.03. A self study program has no instructor. This
  becomes real only if a group or blended program is ever offered.
- Program evaluations, 4.04. Feature 025.
- Purchased content review responsibilities, 4.06. Nothing is purchased.
- Credit measurement. Feature 022.
- An approval workflow — states, queues, notifications, a reviewer inbox. The
  reviewer is a recorded fact on a course, not a role with a task list. Build
  the record; the workflow can come later if two people are ever actually
  passing courses back and forth.
- The LLM content pipeline itself. But read the next section before deciding
  this feature can wait for it.

## The locators this feature is built against
Read these in `docs/2026-Statement-on-Standards-for-CPE-Programs.pdf` before
starting. Quote them into COMPLIANCE.md in the Standard's own words; do not work
from this summary.

- **4.01** — activities, materials, and delivery systems that are current,
  accurate, and effectively designed. Course documentation must contain the most
  recent publication, revision, or review date. Subjects that undergo frequent
  change must be reviewed and revised by a subject matter expert at least once a
  year; other courses at least every two years.
- **4.01.1** — learning activities must be developed by subject matter
  expert(s), competent and current in the subject matter and skilled in the
  appropriate instructional strategies and technology. **If technology is used
  in the development of the program, the content developer is responsible for
  reviewing the content for accuracy.**
- **4.02** — learning activities must be reviewed by content reviewers other
  than those who developed the program, to ensure it is accurate, current, and
  addresses the stated learning objectives. These reviews must occur before the
  first presentation and again after each significant revision. At least one
  licensed CPA in good standing must participate in the development of every
  program in accounting and auditing; at least one licensed CPA, tax attorney,
  or IRS enrolled agent for every program in the field of study of taxes.
- **4.02.1** — reviewers must be qualified in the subject matter. The review is
  a quality control procedure. Where advance review is impractical in rare
  circumstances, the basis for the lack of content review must be documented.

## Build this before the pipeline
4.01.1's last sentence is the reason this feature is scheduled here rather than
after the LLM work, and it is worth reading twice: if technology is used in
developing the program, the content developer is responsible for reviewing the
content for accuracy.

That sentence is about abacadaba specifically. A generation pipeline that exists
first will be designed around one person clicking through a draft, and the
second signature becomes something bolted on afterwards — at which point the
honest answer to "who developed this" is a model, and there is no field to put
it in. Building the chain first means the pipeline has to produce a course that
already has a named human developer and a different named reviewer, because a
course without them will not publish.

## Subject matter experts are not users
The `subject_matter_experts` table has no foreign key to `users`, and should not
grow one.

Three reasons, in order of how much they will matter later:

1. The reviewer on a real CPE program is frequently an outside CPA who has no
   reason to ever log into the admin tool. Requiring an account to be recorded
   as a reviewer means creating dormant accounts to satisfy a data model, which
   is both a security posture problem and a lie about who has access.
2. What the Standard wants documented is competence — credentials, license
   status, jurisdiction. A `users` row carries none of that and should not start
   to.
3. The two records answer different questions with different lifetimes. Auth
   answers "may this person edit this course today". The SME record answers "was
   this person qualified to sign off on this course in August 2026", and it must
   remain true and readable years after the person's account is gone.

A person can obviously have both records. The SME record is the one an audit
reads.

## Data model

New `subject_matter_experts`:
- id, name (not null)
- credentials (string, not null) — the human-readable line, e.g.
  "CPA, active, NH #12345"
- affiliation (string, nullable), bio (text, nullable)
- is_licensed_cpa (bool, not null, default false)
- is_tax_attorney (bool, not null, default false)
- is_enrolled_agent (bool, not null, default false)
- license_jurisdiction (string, nullable)
- created_at

The three booleans map one-to-one onto the sentence in 4.02 rather than onto a
tidier abstraction. Accounting and auditing needs the first; taxes needs any of
the three. A single enum would force a person who is both a CPA and an enrolled
agent into one box, and a derived "is qualified" column would drift from the
rule it was derived from.

New columns on `courses`:
- developer_id (FK subject_matter_experts, nullable, indexed)
- reviewer_id (FK subject_matter_experts, nullable, indexed)
- reviewed_at (timestamptz, nullable)
- review_notes (text, nullable)
- content_updated_at (timestamptz, not null, server default now())
- review_cycle (string, not null, default 'biennial', CHECK in
  ('annual', 'biennial'))

Plus a CHECK that the two experts differ:
`reviewer_id IS NULL OR developer_id IS NULL OR reviewer_id <> developer_id`.
Autogenerate will not write it; add it by hand alongside the 020 constraints.

New `sources`:
- id, course_id (FK, ondelete CASCADE, indexed), position (int, unique per
  course), citation (string, not null), url (string, nullable), retrieved_on
  (date, nullable)

Reuse the two-pass renumber already written for questions and objectives. This
would be the third implementation of it; there should still be one.

## Sources are recorded, not required
Publish validation does not require a source. No locator demands a citation
list, and 4.05.3 is about the opposite concern — that a program must be built on
materials developed for instructional use rather than on third-party material.

But the source list renders in the review panel, directly beside the reviewer
and review-date controls. A reviewer signing off on a draft that cites nothing
should have to notice that this is what they are doing. That is a placement
decision, not a validation rule, and it is the right shape for something the
Standard leaves to the reviewer's judgment.

## content_updated_at, and the bug this feature will produce
The interesting rule here is that a review is only meaningful if it happened
after the content it reviewed. 4.02 says reviews must occur before the first
presentation and again after each significant revision. So:

`content_updated_at` is bumped by every mutation that changes what a participant
would see — course fields, objectives, lessons, questions, choices, video,
thumbnail — through **one helper in `app/services/admin_content.py`**, called
from each write path. Not sprinkled inline; one choke point, so the answer to
"does this mutation count" is in one file.

It is **not** bumped by: setting the developer or reviewer, setting reviewed_at,
writing review notes, adding or editing sources, publishing, or unpublishing.

That exclusion is the whole feature working or not working. If recording a
review bumps the content timestamp, then `reviewed_at < content_updated_at` is
true the instant after every review, publish refuses forever, and the failure
looks like a validation bug rather than a design one. Write the test that
asserts recording a review leaves `content_updated_at` untouched before writing
the helper.

On "significant revision": the Standard says significant, and nothing here can
judge significance. This feature treats every content edit as potentially
significant. That is deliberately over-strict — re-recording a review is a few
seconds of work, and a false positive costs nothing while a false negative is
the thing the Standard exists to prevent.

## Validation rules
These join the 019 and 020 rules in `validate_for_publish`, which must keep
returning every failure at once so feature 017's checklist still works.

1. `developer_id` set.
2. `reviewer_id` set.
3. `reviewer_id != developer_id` — 4.02, stated directly. The database CHECK is
   a backstop; the validator produces the readable message.
4. `reviewed_at` set.
5. `reviewed_at >= content_updated_at`. Message: this course has changed since
   it was reviewed.
6. If `field_of_study` is tagged as accounting or auditing: the developer or the
   reviewer must have `is_licensed_cpa`.
7. If `field_of_study` is Taxes: the developer or the reviewer must have at
   least one of the three credentials.

Rules 6 and 7 need the fields-of-study constant from feature 020 to carry a
credential tag. Put the tag next to the field in
`app/constants/fields_of_study.py`, not in a second list keyed by field name —
two lists NASBA controls will drift, which was already the reasoning behind
020's `/meta/fields-of-study` endpoint.

Note what rule 5 buys on the disclosure side: because publish refuses a review
older than the content, `reviewed_at` on a published course is always the most
recent of the two dates. So 4.01's "most recent publication, revision, or review
date" is just `reviewed_at`, with no max() over three columns and no way for the
displayed date to be wrong. The validation rule makes the disclosure trivially
correct. Do not compute a separate documentation date.

## The gap this feature does not close
A published course that is then edited presents a real problem, and the answer
here is partial.

Editing a published course bumps `content_updated_at`, so re-publishing is
blocked until it is reviewed again. But the course stays published and keeps
serving in the meantime, because unpublishing it would pull a program out from
under participants who are partway through it — which is worse for them and
arguably worse for the sponsor.

The consequence is that a published course can serve edited, unreviewed content
until someone gets to it. The real fix is a draft-and-version model where edits
land on an unpublished revision, and that is a larger feature than this one.

**Record this in COMPLIANCE.md's Gap column against 4.02, explicitly, with the
mitigation** — the admin surface flags the state, and 026's dashboard will
report on it. Do not write a Gap column entry that implies it is handled. This
is exactly the kind of thing abacadaba exists to find before superCPE has an
audit.

## Backend tasks
1. `app/models/subject_matter_expert.py`, `app/models/source.py`, and the six
   columns on `Course`. Then autogenerate the migration and add the CHECK
   constraints by hand. Verify `downgrade -1`.
2. Unlike feature 019, the databases may have rows this time. `content_updated_at`
   is not null — give it a server default in the migration so existing courses
   get a value, and check whether any course is currently published. If one is,
   it will fail rules 1–4 the next time anyone touches publish; say so in the
   changelog rather than letting it surprise someone.
3. SME CRUD under `require_admin`.
4. Source CRUD nested under a course, reusing the existing renumber.
5. The `content_updated_at` helper and its call sites.
6. Credential tags on the fields-of-study constant, and whatever
   `/meta/fields-of-study` needs to expose them.
7. `validate_for_publish` rules 1–7.
8. `GET /courses/{slug}` carries the review date and the developer's and
   reviewer's names and credentials. 4.01 says course documentation rather than
   descriptive materials, so the public page is a choice rather than a
   requirement — make it, because it is a quality signal and the credential line
   is the thing a participant would actually want to see. Bio and affiliation
   are internal.
9. Tests:
   - publishing with no developer is refused, and the message says so
   - publishing with the same person as developer and reviewer is refused
   - publishing with a review older than the last content edit is refused
   - recording a review does not change `content_updated_at`
   - editing a question does change it
   - adding a source does not change it
   - an accounting course with no licensed CPA on either side is refused
   - a taxes course with an enrolled agent as reviewer publishes
   - a non-technical course with neither is unaffected by rules 6 and 7
   - the public course payload carries the review date and both names
   - sources come back in position order and reorder correctly
   - the leak test still passes

## Frontend tasks
1. An SME admin page: list, create, edit. Plain CRUD, no cleverness.
2. A Review panel on the course editor — developer select, reviewer select,
   review date, notes — with the source list rendered beside it per the
   placement decision above.
3. The stale-review state on a published course, stated where the author is
   looking rather than only in the publish checklist.
4. The publish checklist should pick up rules 1–7 automatically through its
   existing `publishErrors` list. Verify that; do not build a second surface for
   it. Feature 020 found the same thing and it held.
5. Public course page renders the last-reviewed date and the two credited
   experts.

## A thing to check rather than assume
The Review panel is a new panel on the course editor, and the course editor now
has one save for the whole page. It must join that batch. Do not give it its own
Save button — that reintroduces exactly the defect 020b spent a feature removing
and 020d spent another feature making legible.

If feature 019a has shipped, the collapsed single-lesson editor needs the panel
too. It is a course-level fact, so it belongs on the course surface in both
layouts.

## Acceptance criteria
- a course cannot be published without a developer, a reviewer, and a review
  date, and the checklist names which is missing
- setting the same expert as both developer and reviewer is refused, in the API
  and by the database
- editing any question, objective, lesson, or course field after a review makes
  publish refuse with a message about the course having changed
- recording the review again clears it, and no other field had to be touched
- an accounting or auditing course refuses to publish unless a licensed CPA is
  named on one side of the chain
- a taxes course accepts a CPA, a tax attorney, or an enrolled agent
- the public course page shows the last-reviewed date, and that date is the
  review date rather than anything computed
- sources can be added, reordered, and deleted, and adding one does not make the
  course's review stale
- an already-published course that is edited shows the stale-review state in the
  editor and keeps serving
- `npm run lint` passes
- pytest passes, with the new tests added

## When done
Append an entry to CHANGELOG.md.

Then append to COMPLIANCE.md. This feature has real rows: 4.01, 4.01.1, 4.02,
and 4.02.1, with the Requirement column in the PDF's own words rather than this
file's paraphrase. One of them — 4.02 — carries a Gap, described above. Write it
plainly.

Then stop.
