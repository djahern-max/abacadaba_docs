# Current Feature

## Feature 018, Lesson thumbnails

## Goal
A lesson has a thumbnail image. It fronts the lesson card and sits behind the
video player before playback, so neither place is a wall of text or a black
rectangle.

## In scope
- A thumbnail image per lesson, stored in Spaces alongside the video
- Admin upload, with guidance on the expected size
- The thumbnail on the lesson card and as the video poster
- A sensible fallback when a lesson has no thumbnail

## Out of scope
- Generating a thumbnail from a video frame. Appealing, but it needs decoding on
  the server or a canvas capture in the browser, and it deserves its own
  feature.
- Multiple sizes, responsive `srcset`, a CDN, or image optimisation
- Replacing the title and description. The thumbnail joins them, it does not
  displace them — a card with no text is worse for scanning and worse for
  screen readers.

## Size
Target 1280x720, the same 16:9 shape as the video player and the same as
YouTube uses. Accept anything 16:9 and reasonably large; show a warning, not an
error, when an upload is a different aspect ratio, since the author may have a
reason.

Accept JPEG, PNG, and WebP. Cap the file at 2 MB — a still image has no business
being larger, and the cap is a cheap guard on an authenticated upload endpoint.

## Data model
Add to lessons:
- thumbnail_key: string, nullable. Null means no thumbnail uploaded.

Mirror the video pattern exactly: key derived from the slug, stored private,
served through a presigned GET. Store as `lessons/<slug>-thumb.<ext>`, keeping
the extension so the content type survives a round trip.

Note the difference from `video_key`: because the extension varies, the key is
not purely derivable from the slug. Persist the whole key rather than
reconstructing it.

## Backend tasks
1. Add `thumbnail_key` to the Lesson model, then
   `alembic revision --autogenerate -m "add thumbnail_key to lessons"`.
   Inspect, upgrade, verify `downgrade -1`.
2. `app/routers/admin_content.py`: `POST /admin/lessons/{id}/thumbnail`, behind
   the existing `require_admin` at router level. Validate content type and size
   before touching storage. Replacing an existing thumbnail overwrites the old
   object rather than accumulating.
3. `GET /lessons/{slug}/thumbnail-url`, mirroring the existing video-url
   endpoint, returning a presigned URL and its expiry. 404 when there is no
   thumbnail, matching how the video endpoint behaves.
4. `LessonSummary` gains a `has_thumbnail` boolean so the list page knows
   whether to fetch a URL, without the list endpoint issuing a presign per
   lesson. Do not put a presigned URL in the list response — they expire, and
   generating one per card on every list request is wasteful.
5. Tests:
   - uploading a thumbnail as an admin sets thumbnail_key
   - uploading as a non-admin returns 403, signed out returns 401
   - an oversized file is rejected
   - a non-image content type is rejected
   - thumbnail-url returns 404 for a lesson without one
   - the leak test still passes

## Frontend tasks
1. `src/api/lessons.js`: `getThumbnailUrl(slug)`.
2. `LessonCard`: render the thumbnail in a 16:9 box above the title, with the
   title and description kept below it. When there is no thumbnail, render a
   plain placeholder block in the same 16:9 box so the grid does not reflow —
   cards jumping height as images load is the usual failure here. Reserve the
   space with `aspect-ratio` rather than waiting for the image.
3. `VideoPlayer`: set the `poster` attribute from the thumbnail URL. This is the
   black-rectangle fix; `poster` is exactly the built-in mechanism for it, so do
   not build an overlay.
4. `AdminLessonEditor`: a thumbnail upload control beside the video one, reusing
   the styled file input from feature 017, showing the current thumbnail when one
   exists. State the recommended 1280x720 in helper text.
5. `alt` text: use the lesson title. A thumbnail is decorative-adjacent but the
   card is a link, and an unlabelled image inside a link is a real accessibility
   problem.

## A note on the presigned URL
Thumbnails are fetched far more often than videos — every card on the list page,
every visit. Presigned URLs expire, so a page left open will eventually show
broken images. Use the same expiry as the video (3600s) for consistency, and
accept that a very stale tab shows placeholders until reload. Do not build
refresh logic for this; it is not worth the complexity here.

If it becomes a problem, the right answer is a public-read prefix in the bucket
for thumbnails only, which is a deliberate decision to make later rather than
something to slip in now.

## Acceptance criteria
- alembic upgrade head adds thumbnail_key, downgrade -1 reverses it
- an admin can upload a thumbnail and see it immediately in the editor
- a non-admin gets 403 on the upload endpoint, a signed out visitor gets 401
- a 3 MB file is rejected with a readable message
- a PDF renamed to .jpg is rejected
- the lesson card shows the thumbnail, with the title and description still
  present
- a lesson without a thumbnail shows a placeholder and the grid does not reflow
  as images load
- the video shows the thumbnail before playback instead of a black rectangle
- replacing a thumbnail shows the new one without a hard refresh
- pytest passes, including the leak test

## When done
Append an entry to CHANGELOG.md and stop.
