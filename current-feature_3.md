# Current Feature

## Feature 003, Video storage and playback

## Goal
A lesson has a video living in DigitalOcean Spaces, and it plays in the browser
via a short lived presigned URL. The Space stays private.

## In scope
- video_key column on lessons, with a migration
- Spaces client service using boto3
- A presigned URL endpoint the player calls
- A minimal upload endpoint guarded by a shared secret
- A VideoPlayer component replacing the placeholder on the detail page

## Out of scope
- Questions, quizzes, scoring, confetti, certificates
- Real auth or user accounts, an admin UI, transcoding, thumbnails
- Captions, playback progress tracking, resume where you left off

## Setup expected before you start
The Space exists and is private. backend/.env has real values for SPACES_KEY,
SPACES_SECRET, SPACES_REGION, SPACES_BUCKET, SPACES_ENDPOINT. Add one new var,
UPLOAD_SECRET, to both .env and .env.example, and to Settings in config.py.

## Data model
Add to lessons:
- video_key: string, nullable. The object key within the bucket, e.g.
  "lessons/intro-to-ratios.mp4". Never a full URL, never a signed URL.

Nullable on purpose: a lesson can exist before its video is uploaded.

## Backend tasks
1. Add video_key to the Lesson model, then:
   alembic revision --autogenerate -m "add video_key to lessons"
   alembic upgrade head
   Confirm downgrade -1 reverses it.
2. app/services/storage.py:
   - A module level boto3 S3 client built from settings, created once and reused.
     Use endpoint_url, region_name, and signature_version s3v4.
   - generate_presigned_get(key, expires_in=3600) returning a URL string.
   - upload_fileobj(fileobj, key, content_type) uploading with private ACL.
   - object_exists(key) returning a bool, catching ClientError rather than
     letting a 404 from S3 escape as a 500.
   Wrap credential and connection failures in a clear application error. A
   missing or wrong key should surface as a readable message, not a stack trace.
3. app/schemas/lesson.py: add video_key to LessonDetail. Add a VideoUrlResponse
   schema with url and expires_in.
4. app/routers/lessons.py: GET /lessons/{slug}/video-url
   - 404 if the lesson does not exist or is unpublished
   - 404 with detail "This lesson has no video yet" if video_key is null
   - otherwise 200 with a presigned URL valid for one hour
5. app/routers/admin.py: POST /admin/lessons/{slug}/video
   - Requires header X-Upload-Secret matching settings.UPLOAD_SECRET, else 401.
     Compare with secrets.compare_digest.
   - Accepts a multipart file upload. Reject anything whose content type is not
     video/mp4 or video/webm with a 400.
   - Builds the key as "lessons/{slug}{ext}", uploads it, sets lesson.video_key,
     commits, returns the lesson's video_key.
   Register under /api/v1. Add a comment noting the shared secret is temporary
   and replaced by real auth in feature 008.
6. backend/scripts/upload_video.py: a small CLI taking a slug and a local file
   path, POSTing to the admin endpoint with the secret from .env. Print the
   resulting key. Runnable as: python -m scripts.upload_video <slug> <path>
7. tests/test_video.py. Mock the storage service, do not hit the network:
   - video-url returns 404 when video_key is null
   - video-url returns 200 and a url when video_key is set
   - upload without the header returns 401
   - upload with a wrong secret returns 401
   - upload of a non-video content type returns 400

## Frontend tasks
1. src/api/lessons.js: add getVideoUrl(slug).
2. src/components/VideoPlayer/: takes a slug. On mount, fetch the presigned URL,
   then render a native HTML5 video element with controls and preload="metadata".
   Handle four states: loading, no video yet, playback error, and playing. Do not
   autoplay. The container is 16:9 and matches the placeholder's dimensions.
3. src/pages/LessonDetail/: replace the placeholder with VideoPlayer when
   lesson.video_key is set, and keep the existing placeholder when it is null.
4. Presigned URLs expire. If the video element fires an error after the URL has
   been held a while, show a "Reload video" button that refetches the URL rather
   than leaving a dead player.

## Acceptance criteria
- alembic upgrade head adds video_key, downgrade -1 reverses it
- python -m scripts.upload_video intro-to-ratios ./sample.mp4 succeeds and the
  object is visible in the DigitalOcean console
- curl on /api/v1/lessons/intro-to-ratios/video-url returns a signed URL, and
  pasting that URL into a browser plays or downloads the file
- the same object's plain unsigned URL returns AccessDenied, proving the Space
  is private
- a lesson without a video still returns 404 on video-url and still shows the
  placeholder
- the video plays inline on the lesson detail page
- upload with a bad secret returns 401
- pytest passes

## When done
Append an entry to CHANGELOG.md and stop.
