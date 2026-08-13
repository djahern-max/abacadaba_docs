# Current Feature

## Feature 008, Accounts, real auth, and progress

## Goal
A user can register, sign in, and see their own history. Attempts made while
signed in belong to that user, and the certificate carries their account name
rather than a typed one. The admin upload secret is replaced by real
authorization.

## In scope
- users table and a user_id on attempts, with migrations
- Registration, login, logout, and a current user endpoint using httpOnly
  session cookies
- Attempts linked to the signed in user when there is one
- Certificates using the account name for signed in users
- A progress page listing the user's attempts
- Replacing the X-Upload-Secret guard with an is_admin check
- Tests

## Out of scope
- Password reset, email verification, OAuth or social login
- Roles beyond a single is_admin boolean, teams, organizations
- An admin UI. The upload CLI stays a CLI, just authenticated differently.
- Merging attempts made anonymously into an account after signing in

## Deliberate decisions
Sessions are httpOnly, SameSite=Lax, Secure in production cookies holding an
opaque session id. Not a JWT in localStorage, which any injected script can
read. Sessions are stored in a database table so logout genuinely revokes.

Anonymous attempts keep working exactly as they do now. The quiz must not sit
behind a signup wall. user_id on attempts is nullable and stays that way.

## Data model
Table users:
- id: integer primary key
- email: citext or a lowercased string, unique, indexed, not null
- password_hash: string, not null
- display_name: string, not null, 2 to 80 characters
- is_admin: boolean, not null, default false
- created_at, updated_at: timezone aware UTC, server defaults

Table sessions:
- id: string primary key, the opaque token, generated with
  secrets.token_urlsafe(32)
- user_id: integer, foreign key to users.id, not null, indexed,
  ondelete CASCADE
- expires_at: timezone aware UTC, not null
- created_at: timezone aware UTC, server default

Add to attempts:
- user_id: integer, foreign key to users.id, nullable, indexed,
  ondelete SET NULL

Session lifetime is 30 days. Define it as a constant, not an inline number.

## Backend tasks
1. Add passlib[bcrypt] to requirements.txt. Hash with bcrypt. Never store or log
   a plaintext password. No other new dependency.
2. app/models/user.py and app/models/session.py, plus user_id on Attempt.
   Import both in app/models/__init__.py.
3. Two migrations, or one if autogenerate produces a clean single file:
   alembic revision --autogenerate -m "create users and sessions"
   Inspect before applying. Confirm the unique index on email and both foreign
   keys. Then upgrade head and verify downgrade -1.
4. app/schemas/auth.py:
   - RegisterRequest: email as EmailStr, password of at least 10 characters,
     display_name 2 to 80 characters
   - LoginRequest: email, password
   - UserPublic: id, email, display_name, is_admin. Never password_hash.
5. app/services/auth.py:
   - hash_password and verify_password wrapping passlib
   - register(db, email, password, display_name), lowercasing the email and
     signalling a conflict if it is taken
   - authenticate(db, email, password) returning the user or None. Compare the
     password even when no user matches, so response timing does not reveal
     whether an email is registered.
   - create_session(db, user), get_session_user(db, token) which returns None
     for a missing or expired session, and delete_session(db, token)
   - a purge helper deleting expired sessions, called opportunistically on login
6. app/dependencies.py:
   - get_current_user(request, db) reading the session cookie and returning the
     user or None
   - require_user raising 401 when there is no user
   - require_admin raising 401 when there is no user and 403 when the user is
     not an admin. The distinction matters.
7. app/routers/auth.py:
   - POST /auth/register, 201 with UserPublic, sets the session cookie,
     409 if the email is taken
   - POST /auth/login, 200 with UserPublic, sets the cookie, 401 on bad
     credentials with a message that does not reveal which field was wrong
   - POST /auth/logout, 204, deletes the session row and clears the cookie
   - GET /auth/me, 200 with UserPublic or 401
   Cookie settings come from config so production can set Secure and a domain.
   Add SESSION_COOKIE_SECURE to .env.example, defaulting to false locally.
8. Attempts: start_attempt takes an optional user and sets user_id. The route
   uses get_current_user, not require_user, so anonymous attempts still work.
9. Certificates: when the attempt has a user, use that user's display_name as
   recipient_name automatically and do not require one in the request body. When
   the attempt is anonymous, keep the current typed name behavior. Update the
   comment written in feature 007 to say this is now done. Reword the
   verification page copy so a certificate from a signed in user is described as
   issued to an account holder rather than self reported, and keep the old
   wording for anonymous ones. CertificateVerification gains a boolean saying
   which it is.
10. app/routers/attempts.py: GET /me/attempts behind require_user, returning the
    signed in user's completed attempts newest first, each with lesson title and
    slug, score, passed, completed_at, and certificate_code when present.
11. Replace the X-Upload-Secret guard on the admin upload route with
    require_admin. Remove UPLOAD_SECRET from config, .env, and .env.example.
    Update scripts/upload_video.py to log in with an admin email and password
    read from the environment and reuse the session cookie.
12. backend/scripts/make_admin.py: a CLI taking an email and setting is_admin
    true, so the first admin can exist. Runnable as
    python -m scripts.make_admin <email>
13. tests/test_auth.py:
    - registering returns 201 and sets a cookie
    - registering a duplicate email returns 409, and case insensitively so
    - a short password returns 422
    - login with correct credentials returns 200 and sets a cookie
    - login with a wrong password returns 401
    - /auth/me without a cookie returns 401
    - /auth/me with a cookie returns the user, and never includes password_hash
    - logout clears the session so /auth/me returns 401 afterward
    - an expired session returns 401
14. tests/test_admin_auth.py:
    - upload with no session returns 401
    - upload as a non admin user returns 403
    - upload as an admin succeeds, with storage mocked
15. Update existing tests for attempts and certificates where behavior changed.
    Anonymous flows must still pass unchanged.

## Frontend tasks
1. src/api/auth.js: register, login, logout, getMe. Every request in
   src/api/client.js must send credentials: "include" so the cookie travels.
   CORS on the backend needs allow_credentials true and an explicit origin,
   not a wildcard, or the browser will silently drop the cookie.
2. src/context/AuthContext.jsx: holds user and a loading flag, calls getMe once
   on mount, exposes login, register, and logout. Wrap the app in main.jsx.
3. src/components/Header/: extract the header out of App.jsx. Show Sign in and
   Register links when signed out, and the display name, a My progress link, and
   Sign out when signed in.
4. src/pages/Login/ and src/pages/Register/ with their own CSS Modules. Inline
   validation, a disabled button while submitting, and a clear error on failure.
   On success, redirect to where the user came from, defaulting to "/".
5. src/pages/Progress/: route "/me", listing the user's attempts with lesson
   title, score, pass or fail, date, and a download link when a certificate
   exists. Handle the empty case with a prompt to take a quiz. Redirect to login
   when signed out.
6. src/pages/Result/: when signed in, skip the name form entirely. Claim the
   certificate with the account name and show the code and download link
   directly. Remove the localStorage persistence added in feature 007 and drive
   the claimed state from the API response instead, since the attempt now knows
   whether it has been claimed.
7. Register all new routes in App.jsx.

## Acceptance criteria
- alembic upgrade head creates users and sessions and adds attempts.user_id,
  downgrade -1 reverses
- curl registering returns 201 with a Set-Cookie header
- curl /auth/me with that cookie returns the user and no password_hash anywhere
  in the response
- curl /auth/me after logout returns 401
- python -m scripts.make_admin you@example.com promotes an account
- video upload with no session returns 401, as a normal user returns 403, and as
  an admin succeeds
- taking a quiz signed out still works and still produces a certificate with a
  typed name
- taking a quiz signed in shows no name form and produces a certificate carrying
  the account display name
- /me lists that attempt with its score and a working certificate download
- signing out and revisiting /me redirects to login
- pytest passes, including the leak, replay, and certificate tests

## When done
Append an entry to CHANGELOG.md and stop.
