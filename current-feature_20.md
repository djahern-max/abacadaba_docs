# Current Feature

## Feature 020, Program metadata and learning objectives

## Goal
A course carries the descriptive information the Standards require a
participant to be able to read before they decide to take it: what they will be
able to do afterwards, who it is pitched at, what they need to know first, and
what subject area it counts toward. All of it is visible on the public course
page, not just in the admin editor.

## In scope
- Learning objectives as ordered rows on a course
- Program knowledge level, prerequisites, advance preparation
- Field of study, from NASBA's published list
- Publish validation enforcing the conditional rules these fields carry
- Disclosure of all of it on the public course page, before starting

## Out of scope
- The development and review chain, and the "most recent publication, revision,
  or review date" that 4.01 requires. Feature 021 produces that date; adding a
  column for it here would mean adding a column nothing writes to.
- Refund, cancellation, and complaint resolution policies. 8.01.1 requires these
  published too, but they are site-wide static pages rather than course fields.
  Feature 026.
- Credit calculation and field-of-study credit splits. Feature 022.
- Objectives on individual lessons. See "Objectives belong to the course" below.

## The locators this feature is built against
Read these in docs/2026-Statement-on-Standards-for-CPE-Programs.pdf before
starting. Quote them into COMPLIANCE.md; do not work from this summary.

- 3.01 — activities based on learning objectives that articulate the
  professional competence participants should achieve
- 3.01.1 — knowledge level, content, and objectives specified so a potential
  participant can judge fit. Levels: Basic, Intermediate, Advanced, Update,
  Overview
- 3.02.1 — Intermediate, Advanced, and Update programs must clearly identify
  prerequisite education, experience, and advance preparation in precise
  language. Basic and Overview note them if applicable, otherwise state "none"
- 8.01.1 — significant features disclosed in advance
- 8.01.2 — prerequisites and advance preparation identified in the descriptive
  materials

This is the first feature where COMPLIANCE.md has real rows to write. Take the
Requirement column from the PDF's own words.

## Objectives belong to the course
Objectives attach to the program a person completes, which is the course. A
lesson is a segment inside it.

There is a real argument for per-lesson objectives as an instructional design
practice, and superCPE may want them later. Do not build it now. Two levels of
objectives means deciding how they roll up, which of them publish validation
checks, and which appear on a certificate — questions with no forced answer yet.

## Data model
New `learning_objectives` table:
- id, course_id (FK, ondelete CASCADE, indexed), position (int, unique per
  course), text (string, not null)

Ordered rows rather than a text blob. They are validated individually, counted,
and will be reused in feature 022's course documentation; a newline-delimited
textarea would have to be parsed back apart every time.

Add to `courses`:
- program_level: string, not null. One of basic, intermediate, advanced,
  update, overview.
- field_of_study: string, not null.
- prerequisites: text, nullable
- advance_preparation: text, nullable

On the column type for the two enumerated fields: use a plain string with a
CHECK constraint and a Python constant holding the allowed values, not a native
Postgres enum. The lists come from documents NASBA revises on its own schedule,
and altering a native enum in a migration is far more friction than editing a
constant and a constraint.

## Field of study values
Enumerate them once, in `app/constants/fields_of_study.py`, from the January
2024 Fields of Study document in docs/. Do not type them from memory and do not
invent a shortened list. Technical: Accounting, Accounting (Governmental),
Auditing, Auditing (Governmental), Business Law, Economics, Finance,
Information Technology, Management Services, Regulatory Ethics, Specialized
Knowledge, Statistics, Taxes. Non-technical: Behavioral Ethics, Business
Management & Organization, Communications and Marketing, Computer Software &
Applications, Personal Development, Personnel/Human Resources, Production.

Verify that list against the PDF rather than trusting this file.

Carry the technical/non-technical distinction alongside each value — state
boards limit non-technical credit, so superCPE will need it, and recording it
now costs one column in a constant.

Add one more value, `non_cpe`, labelled "Not CPE eligible", and make it the
default for new courses. abacadaba's content is deliberately non-financial and
none of it belongs in a real field of study. The sentinel keeps the column
honest and non-nullable instead of leaving it blank and meaningless.

## Validation rules
These go in `validate_for_publish` on the course, alongside feature 019's
rules, and must keep returning every failure at once so feature 017's checklist
still works.

1. At least one learning objective, each with non-whitespace text.
2. `program_level` set to one of the five values.
3. `field_of_study` set.
4. If `program_level` is intermediate, advanced, or update: `prerequisites` and
   `advance_preparation` must both be non-empty. This is 3.02.1 stated
   directly, and it is the interesting rule in this feature — a conditional
   requirement driven by another field's value.
5. If `program_level` is basic or overview: both may be empty, and the public
   page renders "None" rather than hiding the row. Stating "none" explicitly is
   what 3.02.1 asks for; an absent row is not the same as a stated none.

Do not validate objective text against a verb list. Measurable-verb phrasing is
a quality expectation, not something to enforce with a regex, and a bad list
would block legitimate wording. It belongs in the reviewer's judgment in
feature 021.

## Backend tasks
1. `app/models/learning_objective.py` plus the four columns on `Course`. Then
   `alembic revision --autogenerate -m "add program metadata and learning
   objectives"`. Autogenerate will not write the CHECK constraints — add them by
   hand. Verify `downgrade -1`.
2. `app/constants/fields_of_study.py` and `app/constants/program_levels.py`.
3. `app/services/admin_content.py`: objective CRUD with the same move-up and
   move-down reordering pattern already used for questions, including the
   two-pass renumber that dodges the position unique constraint. Reuse it; do
   not write a second implementation.
4. `app/routers/admin_content.py`: objective routes under a course, and the new
   fields on the course update payload. Behind the existing router-level
   `require_admin`.
5. `GET /courses/{slug}` returns objectives, level, field of study,
   prerequisites, and advance preparation in the public payload. This is the
   disclosure requirement — if it is only in the admin API, the feature is not
   done.
6. `GET /meta/fields-of-study` and `/meta/program-levels`, so the admin editor
   populates its selects from the server rather than duplicating the lists in
   JavaScript. Two copies of a list NASBA controls will drift.
7. Tests:
   - publishing with no objectives is refused, and the message says so
   - publishing an intermediate course with empty prerequisites is refused
   - publishing a basic course with empty prerequisites succeeds
   - an unknown field of study value is rejected by the constraint
   - objectives come back in position order and reorder correctly
   - the public course payload carries all five pieces of metadata
   - the leak test still passes

## Frontend tasks
1. `CourseDetail`: a "What you will learn" list of objectives, and a details
   block showing level, field of study, prerequisites, and advance preparation.
   Render "None" for empty prerequisites and advance preparation rather than
   omitting the rows.
2. Place all of it **above** the assessment button and the lesson list. It is
   pre-enrolment disclosure; below the fold after the content is not disclosure
   in advance.
3. `AdminCourseEditor`: an objectives editor reusing the question editor's
   add/edit/reorder interaction, plus selects for level and field of study fed
   by the meta endpoints, plus two textareas.
4. The selects show the technical/non-technical grouping as optgroups, with
   "Not CPE eligible" first and selected by default.
5. Publish checklist picks up the new failures automatically from the existing
   validation response. Confirm this rather than adding checklist entries by
   hand — if it needs manual entries, feature 017's design has a gap worth
   knowing about.

## Acceptance criteria
- `alembic upgrade head` adds the table and columns with working CHECK
  constraints; `downgrade -1` reverses
- an admin can add, edit, reorder, and delete learning objectives
- the field of study select lists every value from the NASBA document, grouped
  technical and non-technical, defaulting to Not CPE eligible
- an intermediate course cannot publish without prerequisites and advance
  preparation, and the checklist says which is missing
- a basic course publishes with both blank, and its public page shows "None"
- the public course page shows objectives, level, field of study,
  prerequisites, and advance preparation above the assessment button
- a signed-out visitor sees all of it without starting anything
- pytest passes, including the leak test

## When done
Append an entry to CHANGELOG.md.

Then append to COMPLIANCE.md: one row per locator this feature satisfies, with
the Requirement column quoted from
docs/2026-Statement-on-Standards-for-CPE-Programs.pdf. Expect rows for 3.01,
3.01.1, 3.02.1, 8.01.1, and 8.01.2 — but verify each against the PDF and drop
any that this feature does not actually meet. 8.01.1 in particular is only
partly satisfied here, since it also requires published refund, cancellation,
and complaint resolution policies; record that in the Gap column rather than
marking it done.

Then stop.
