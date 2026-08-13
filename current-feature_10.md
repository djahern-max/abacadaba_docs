# Current Feature

## Feature 010, Admin authoring

## Goal
An admin can create a lesson, upload its video, write its five questions, and
publish it, entirely in the browser. No CLI, no seed script, no deploy.

## In scope
- Admin CRUD for lessons, questions, and choices
- Browser video upload replacing the CLI
- Publish and unpublish, with validation that blocks publishing broken lessons
- An admin section in the frontend behind require_admin
- Tests

## Out of scope
- Rich text editing, markdown in descriptions, image uploads
- Drag and drop reordering. Use simple move up and move down.
- Bulk import, CSV, duplication of lessons
- Question banks larger than the five that get served
- Draft previews for non admins

## Measure this while you build
Content operations cost is the number this whole experiment exists to learn.
When the feature is done, time yourself authoring one complete lesson from an
empty form to published, and record the number in CHANGELOG.md alongside the
entry. It is data, not vanity.

## Validation rules, enforced server side
A lesson can be published only when all of these hold:
- title, slug, and description are non empty
- video_key is set
- it has exactly five questions
- every question has at least two choices and exactly one correct choice
Return a 422 listing every failed rule at once, not just the first. The admin
should see the whole list, not play whack a mole.

An unpublished lesson is invisible to the public endpoints, which already
behave this way. Do not weaken that.

## Backend tasks
1. app/schemas/admin.py: request and response schemas for lessons, questions,
   and choices as the admin sees them. The admin choice schema does include
   is_correct, since an admin must see and set it. Name these AdminChoice,
   AdminQuestion, AdminLesson so they can never be confused with the public
   leak free schemas. Add a comment on ChoicePublic pointing at its admin twin,
   so nobody later "simplifies" them into one class.
2. app/services/admin_content.py:
   - lesson create, update, delete, list including unpublished
   - question create, update, delete, and reorder within a lesson
   - choice create, update, delete, and reorder within a question
   - set_correct_choice(db, question_id, choice_id) which clears the flag on
     the question's other choices in the same transaction, so two correct
     answers cannot coexist even briefly
   - validate_for_publish(db, lesson) returning a list of failure messages
   - publish and unpublish, refusing to publish when the list is non empty
   Reordering rewrites the position column for the whole set rather than
   swapping two rows, to avoid unique constraint collisions mid update.
3. app/routers/admin.py, everything behind require_admin:
   - GET, POST /admin/lessons and GET, PATCH, DELETE /admin/lessons/{id}
   - POST /admin/lessons/{id}/publish and /unpublish
   - POST /admin/lessons/{id}/questions, PATCH and DELETE
     /admin/questions/{id}, POST /admin/questions/{id}/move
   - POST /admin/questions/{id}/choices, PATCH and DELETE
     /admin/choices/{id}, POST /admin/choices/{id}/move
   - POST /admin/questions/{id}/correct-choice
   - keep the existing video upload route, now also behind require_admin
   Deleting a lesson with completed attempts must not orphan data. Either
   refuse with a 409 explaining why, or soft delete. Choose refusal, it is
   simpler and safer.
4. Slug handling: generate a slug from the title on create if none is given,
   lowercase and hyphenated. Reject a duplicate slug with 409. Changing a slug
   on a published lesson breaks existing links, so warn in the API response
   description and require the change to be explicit.
5. Retire scripts/seed.py from routine use. Leave the file, add a comment at the
   top saying content is now authored in the admin UI and this exists only for
   bootstrapping a fresh database.
6. tests/test_admin_content.py:
   - a non admin gets 403 on every admin route, an anonymous request gets 401
   - creating a lesson returns it unpublished
   - publishing a lesson with four questions returns 422 listing the failure
   - publishing a lesson whose question has no correct choice returns 422
   - publishing a complete lesson succeeds and it then appears in the public list
   - unpublishing removes it from the public list
   - setting a correct choice clears the previous one
   - reordering questions produces contiguous positions starting at 1
   - deleting a lesson with a completed attempt returns 409

## Frontend tasks
1. src/api/admin.js covering every admin endpoint.
2. src/pages/Admin/ with nested routes under /admin, all redirecting to login
   when signed out and showing a plain "not authorized" page for signed in non
   admins. Never render an admin link for a non admin.
   - /admin: lesson list showing published state, question count, and whether a
     video exists. A create button.
   - /admin/lessons/:id: the editor.
3. The lesson editor holds three sections: details, video, and questions.
   - Details: title, slug, description, duration. Save is explicit, not
     autosave, so a half typed field never persists.
   - Video: current video if present, and a file input that uploads with a
     visible progress indicator. A 200MB upload with no feedback feels broken.
   - Questions: each question inline editable with its choices, a radio group
     selecting the correct one, add and remove buttons, and move up and down.
4. A publish button showing the validation failures returned by the server as a
   checklist, so the admin can see exactly what is missing. Disable it while the
   list is non empty, but still render the list.
5. Add an Admin link to the header, visible only when user.is_admin.
6. Every admin page gets its own CSS Module. This is an internal tool, so favour
   density and clarity over polish, but do not let it look broken.

## Acceptance criteria
- a non admin visiting /admin sees a not authorized page, not a crash
- an admin can create a lesson, upload a video with visible progress, add five
  questions with four choices each, mark the correct ones, and publish
- the published lesson appears on the public list and is fully playable and
  quizzable end to end without touching a terminal
- publishing an incomplete lesson shows the checklist of what is missing
- selecting a different correct choice unsets the previous one
- moving a question up reorders it and the order survives a refresh
- unpublishing hides the lesson from the public list but keeps it in the admin list
- deleting a lesson with attempts is refused with an explanation
- the leak test still passes, and the public quiz response still contains no
  is_correct
- pytest passes

## When done
Append an entry to CHANGELOG.md, including the measured time to author one
lesson, and stop.
