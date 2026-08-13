# Current Feature

## Feature 017, Authoring UX

## Goal
Authoring a lesson stops tripping the author. Duration fills itself in, the form
stops silently disagreeing with the server, and the publish requirements are
visible before you press publish rather than after.

## In scope
- Reading video duration from the file instead of asking for it
- Making unsaved changes visible, and making publish validation honest
- Styling the file input
- A non-blocking upload with a clear status
- An always-visible publish checklist

## Out of scope
- Real background/queued uploads with a job runner. See the note below.
- Video transcoding, multiple resolutions, a CDN
- Thumbnails. That is feature 018.
- Changing what `validate_for_publish` requires. The rules are fine; only their
  visibility is the problem.

## Where these came from
Every item here was hit while authoring the first real lesson. None of them are
speculative.

## Part 1, duration reads itself
`duration_seconds` predates video entirely — feature 002 added it so the lesson
card could print "5 min". Feature 011 then quietly promoted it into the
denominator of the watch gate, and nobody revisited the fact that a human still
types it. A wrong value silently breaks the gate; a blank one disables it.

Fix it in the browser, not the server. On file selection, create an object URL,
read `video.duration` from the `loadedmetadata` event, revoke the URL, and fill
the duration field with the value rounded **down**. Rounding up sets a target
past the end of the file, which can never be satisfied.

Leave the field editable and mark it as auto-filled. Do not add ffmpeg or
ffprobe to the container — that is a system binary, a larger image, and more
attack surface, against the posture HARDENING.md sets out. Client-reported
duration is untrusted, but an admin can already type any number they like, so
this weakens nothing.

## Part 2, the form disagreeing with the server
This is the one that cost the most time. The Details form holds unsaved values,
`validate_for_publish` reads the database, and publish reports a missing
description while a description is visibly on screen. The author is looking at
one state and the server is answering about another.

Fix, smallest version that is actually correct:
- Track whether the Details form is dirty.
- While dirty, disable Publish and say why: "Save your details first."
- Warn on navigating away with unsaved changes.

Do not make Publish silently save first. Publishing and saving are different
intentions, and merging them means a stray click publishes edits the author was
still thinking about.

## Part 3, the file input
`<input type="file">` renders as an unstyled native control that ignores the
rest of the design. Use the standard pattern: visually hide the input, wrap it
in a `<label>` styled as a button, and show the selected filename beside it.

Keep it a real `<label for=...>` bound to a real input. Do not rebuild it out of
a div and a click handler — that loses keyboard access and the file dialog.

## Part 4, the upload
The author currently watches a progress bar and can do nothing else.

Be honest about the ceiling here: a true "we'll process it, come back later"
flow needs the bytes to land somewhere durable and a worker to pick them up.
That is a job queue, and it is a bigger feature than this one.

What this feature does instead:
- The upload no longer blocks the rest of the editor. Questions and details stay
  editable while it runs.
- The progress bar is replaced by a status line: uploading with a percentage,
  then "Processing", then a confirmation once the server returns a `video_key`.
- Navigating away mid-upload warns, since leaving does cancel it.

Say plainly in the UI that the upload is still running. Do not imply it will
finish in the background when it will not.

## Part 5, the publish checklist
`validate_for_publish` already returns every failed rule at once — feature 010
built it that way deliberately. The problem is it only speaks when you press
publish.

Show the same checklist permanently in the editor, ticking items off as they are
satisfied: five questions, one correct choice each, video uploaded, title, slug,
description. Then "why can't I publish" is answered before it is asked, and the
draft state stops feeling like a dead end.

Read it from the existing endpoint rather than reimplementing the rules in the
frontend. Two copies of that logic will drift.

## Backend tasks
None expected. If a task here appears to need a backend change, stop and check
whether the existing validation endpoint can be read instead.

## Frontend tasks
1. `AdminLessonEditor`: duration auto-fill on file selection.
2. `AdminLessonEditor`: dirty tracking, disabled Publish with a reason, and an
   unsaved-changes warning.
3. A styled file input, reused by the video upload control.
4. Non-blocking upload with a status line and a mid-upload navigation warning.
5. A persistent publish checklist fed by the existing validation response.

## Acceptance criteria
- selecting a video file fills the duration field with the file's duration,
  rounded down, and the field stays editable
- with unsaved details, Publish is disabled and explains why
- navigating away with unsaved changes warns
- saving then publishing works, and publish never reports a field that is
  visibly filled in
- the file input matches the rest of the design and opens via keyboard
- questions can be edited while an upload is in progress
- leaving mid-upload warns first
- the publish checklist is visible on a fresh draft and ticks items off as they
  are completed
- authoring a complete lesson start to finish requires no guessing about what is
  outstanding
- `npm run lint` passes
- pytest passes, unchanged

## When done
Append an entry to CHANGELOG.md and stop.
