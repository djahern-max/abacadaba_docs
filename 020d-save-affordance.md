# Current Feature

## Feature 020d, The save affordance

## Goal
An author can tell whether their work is saved without being told, and can tell
what will happen when they click Save before they click it.

## In scope
- A button system where disabled looks disabled, on every variant
- The unsaved-changes bar becomes an event rather than a fixture
- Per-field marking so feedback happens where the author is looking
- Cmd/Ctrl+S
- A navigation guard when changes are pending

## Out of scope
- Autosave. 020b decided against it for a reason that still holds, see below.
- Changing the one-save-per-page model. That decision was right; only its
  presentation failed.
- The single-lesson authoring path. Filed separately as 019a, because it amends
  019's hierarchy rather than 020's save model.
- Any schema change, any backend change. This feature is entirely client state
  and CSS.
- What publish validation requires, and the publish checklist's contents.

## Where this came from
An authoring session on 2026-08-18. Ten minutes lost trying to work out how to
save a course description, on a page where the Save button was present, visible,
and doing exactly what it was built to do.

This is not a bug report. Every component behaved as specified. The defect is
that the specification produced a page an author cannot read.

## The diagnosis, read this before changing anything
Look at a screenshot of the course editor before you start. `Add objective` and
`Add lesson` are light purple and enabled. `Save` is light purple and
**disabled**. Three buttons, one treatment, two opposite meanings.

The page teaches the author that light purple means "click me" — twice, in the
two panels directly above Save. So when Save renders in the same colour, it
carries no information at all. There is no way to learn from this page that Save
is waiting on you, because the page is actively teaching the opposite.

Three things compound it:

1. **The status text is as far from the author's eyes as the layout permits.**
   The author is editing near the top-left of a long form. "No unsaved changes"
   sits bottom-left, low contrast, below the fold, phrased as reassurance rather
   than as a state anyone can act on.
2. **Nothing changes near the field that was edited.** Type in the description
   and every pixel within several hundred of the cursor stays identical. All
   feedback happens off-screen.
3. **A bar that is always present and usually says nothing becomes wallpaper.**
   It was scrolled past repeatedly in the session that produced this feature.

Fix the first item and the other two probably stop mattering. Fix all three and
the page explains itself.

## Why not autosave
Because 020b was right. A published course is a document a participant is
working through and that a sponsor has to be able to produce on audit; edits
landing on it keystroke by keystroke is the wrong behaviour for that object
regardless of how convenient it feels while authoring. Explicit save stays.

The problem was never that saving is explicit. It is that "explicit" was
implemented at four granularities (020b), and then the one remaining control was
made invisible (this feature).

## Part 1, the button system

Right now `disabled` is styled per-component, when it is styled at all, and a
disabled primary button reads as an enabled secondary one. Make disabled a
modifier that overrides the variant, defined once.

Three variants, one disabled state:

- **primary** — solid purple, white text. Save, Publish, Unpublish. The action
  the author came to this panel to take.
- **secondary** — light purple, dark text. Add objective, Add lesson, Move up,
  Move down. Supporting actions.
- **danger** — red outline. Delete objective, Delete course.

Then, applying to all three and winning over all three:

- **disabled** — grey background, grey text, `cursor: not-allowed`, no hover or
  active state. Must be visually distinct from every enabled variant, including
  secondary. This is the whole feature in one rule.

Use the real `disabled` attribute, not a class that only looks disabled — a
styled-but-clickable button is worse than either state, and screen readers need
the attribute to announce it.

Do not introduce a fourth variant, a new colour, or a component library. There
is one shared button module already; this is an edit to it, not a replacement.

## Part 2, the bar becomes an event

Today: always mounted, says "No unsaved changes" most of the time.

Change to: **hidden when clean.** It appears when the page becomes dirty and
disappears on successful save. Its arrival is the signal. Keep the transition
short — a slide or fade under 200ms — so it registers as something happening
rather than as chrome that was always there.

When visible it says there are unsaved changes, in a colour that reads as
attention rather than as a status line, and it holds the Save button. Give it
`aria-live="polite"` so the appearance is announced rather than only seen.

It must not block the page, dim the content behind it, or trap focus. It is a
prompt, not a modal.

**A design note worth acting on:** consider putting Save in the page header
beside the title as well, so it is reachable without scrolling on a long form.
If you do, the two must share one state and one handler — two Save buttons that
can disagree is a worse defect than the one being fixed. If that is not clean,
the bar alone is sufficient. Do not ship both if it costs a second source of
truth.

## Part 3, mark the dirty field

Any input whose value differs from the last saved value gets a visible marker —
a left border accent is enough. Feedback then happens where the author is
looking rather than only at the bottom of the page.

This needs the original server values kept alongside the working copy, so the
comparison is per field rather than one page-level boolean. See "A thing to
check" below.

Clear every marker on successful save, not on save click.

## Part 4, keyboard and navigation

- **Cmd+S / Ctrl+S** saves when the page is dirty, and does nothing when clean.
  `preventDefault` so the browser's own save dialog never appears. Bind on the
  editor pages only; do not put a global handler on the app.
- **Navigation guard.** Leaving an editor with pending changes prompts first.
  Use the router's blocker for in-app navigation, and `beforeunload` for tab
  close and reload. `beforeunload` cannot carry custom text in any current
  browser — do not write a message for it and assume it appears.

## A thing to check rather than assume
020b consolidated the editors onto batched client state, but check what that
state actually holds. If it tracks a single `isDirty` boolean rather than a
snapshot of the last saved values, Part 3 needs that snapshot added first, and
Part 2's "disappears on successful save" needs the snapshot refreshed from the
save response rather than from the local working copy. Establish this before
writing any of the four parts; it decides how much of this feature is state
work versus CSS.

Also confirm the upload controls still save on file selection. Video and
thumbnail uploads are not part of the batched save and must not be folded into
it — a 231 MB upload behind a Save button is a different feature and a worse
one. If the bar appears when a file is merely selected, that is a bug.

## Backend tasks
None. If this feature touches a file under `backend/`, something has gone wrong.

## Frontend tasks
1. The shared button module: three variants plus a disabled modifier that wins.
   Audit every existing button for which variant it should be — several are
   currently light purple by default rather than by decision.
2. `AdminCourseEditor` and `AdminLessonEditor`: the bar becomes conditional on
   dirty state, with the transition and `aria-live`.
3. Per-field dirty comparison and the input marker, in whichever module holds
   the batched state from 020b.
4. The keyboard handler, scoped to the editor pages.
5. The navigation guard, both halves.
6. Clear markers and hide the bar on the save response resolving, not on click.

## Acceptance criteria
- With no changes pending, Save is grey, `cursor: not-allowed`, and visibly
  different from `Add lesson` sitting above it
- Typing in the course description makes the bar appear, marks that field, and
  turns Save solid purple
- Clicking Save clears the marker, hides the bar, and the description survives a
  hard refresh
- Cmd+S saves when dirty and does nothing when clean, and never opens the
  browser's save dialog
- Navigating to another admin page with changes pending prompts first
- Closing the tab with changes pending prompts first
- Selecting a video file still uploads immediately and does not make the page
  dirty
- No button anywhere in the admin renders as light purple while disabled
- A screen reader announces the bar when it appears, and announces Save as
  disabled when it is
- pytest passes, unchanged — this feature adds no backend behaviour to test

## When done
Append an entry to CHANGELOG.md.

Then check COMPLIANCE.md. This feature almost certainly maps to no locator. The
Standards govern what a program must contain and disclose, not how an authoring
tool renders a button, and the disclosure requirement that the course
description satisfies was already recorded under 020.

There is one argument the other way, and it is worth writing down only if you
find you believe it: content that silently fails to save is content the sponsor
cannot produce on audit, which touches Section 9's documentation requirements.
That is a stretch — the failure mode here was an author who could not find a
control, not a system that lost committed data. If you cannot quote a locator
that genuinely covers it, write nothing and say so explicitly in your summary.
Do not invent a locator to have something to record.

Then stop.
