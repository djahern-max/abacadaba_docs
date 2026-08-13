# Current Feature

## Feature 013, Sign in with Google

## Goal
A visitor can create an account or sign in with their Google account and end up
in exactly the same session as a password account, with the same cookie and the
same permissions.

## In scope
- Server side OAuth 2.0 authorization code flow against Google
- google_sub on users, and password_hash becoming nullable
- Account linking, on a verified email only
- A sign in with Google button on both Login and Register
- Tests, with the token exchange mocked

## Out of scope
- Any other provider. Apple, GitHub, and Microsoft are the same shape and can
  reuse this once it exists, but adding two providers at once means neither gets
  its edge cases thought through.
- Unlinking Google, or adding a password to an account created through Google.
  Both need a password reset flow, which does not exist yet and deserves its own
  feature.
- Email verification for password accounts. Still missing, still worth doing,
  still not this.
- Refresh tokens, storing Google access tokens, or calling any Google API on the
  user's behalf. Identity is read once, at sign in, and then discarded.
- JWT. Sessions stay opaque and database backed exactly as feature 008 built
  them.

## Read this before starting
Authentication does not change. Google replaces the identity check and nothing
else. Once Google has confirmed who someone is, call the existing
`auth_service.create_session(db, user)` and the existing `_set_session_cookie`.
`require_user`, `require_admin`, `/auth/me`, certificate claiming, and feature
011's user_id watch progress lookup all keep working untouched.

If you find yourself adding a second session mechanism, stop. That is the
mistake this feature exists to avoid.

## The security decision, read this twice
Resolve the identity in this order:

1. Match on `google_sub`. Google's subject id is stable for the life of the
   account. Email is not: people change theirs.
2. If no match, fall back to matching the email against an existing account, and
   link the two **only if the ID token's `email_verified` claim is true**. If it
   is false, refuse and say so.
3. If neither matches, create a new user with `password_hash` NULL.

Step 2 without the `email_verified` check is an account takeover. Anyone able to
create a Google account carrying someone else's email address would inherit that
person's account, including its admin flag. Do not treat the claim as optional
because it is almost always true.

## Data model
Add to users:
- google_sub: string, unique, indexed, nullable

Alter users:
- password_hash: becomes nullable. A Google-created account has no password.

No other schema changes. Sessions are unchanged.

## A trap in authenticate()
`auth_service.authenticate` currently calls `verify_password(password,
user.password_hash)` unconditionally. Against a Google-only account that passes
None into passlib and raises, turning a wrong-credentials 401 into a 500.

Guard it: when `password_hash` is None, verify against `_DUMMY_HASH` and return
None, exactly as the missing-user branch already does. That keeps the response
timing uniform and returns the same "Incorrect email or password" as any other
failure, so the endpoint does not become an oracle for which addresses are
Google-only.

## Dependency
Add `authlib`, pinned, to requirements.txt. One new dependency.

The justification: the callback has to verify the ID token's signature against
Google's JWKS endpoint, whose keys rotate. Hand rolling JWT signature
verification is the single easiest place in this feature to write something that
looks correct and accepts forged tokens. Do not do it with httpx and a JSON
parse.

## Configuration
Add to `app/config.py`, all optional:
- google_client_id: str | None = None
- google_client_secret: str | None = None
- google_redirect_uri: str | None = None

Optional so that a deploy without them configured still boots. When any is
unset, `/auth/google/start` returns 404 and the frontend hides the button, so a
half-configured environment fails visibly rather than at the callback.

Add all three to `backend/.env.example` and to the production `.env.example`.

## Google Cloud Console, do this first
1. Create a project and configure the OAuth consent screen as External.
2. Request `openid`, `email`, and `profile` scopes only. These are non-sensitive,
   so the app does not enter Google's verification review.
3. Create an OAuth 2.0 Client ID of type Web application.
4. Register **both** redirect URIs on the same client:
   - `http://localhost:8000/api/v1/auth/google/callback`
   - `https://api.abacadaba.com/api/v1/auth/google/callback`
   Google exempts localhost from its HTTPS requirement, which is why this can be
   developed locally at all. Match the URIs exactly, trailing slash included:
   a mismatch is the most common failure in this flow and its error message
   names the URI it received, so read it rather than guessing.
5. While the consent screen is unpublished, add your own Google account under
   Test users or sign in will be refused.

The client secret goes in `.env` only. Never in a build arg, never committed.

## Backend tasks
1. `requirements.txt`: add authlib, pinned.
2. Add `google_sub` to the User model and make `password_hash` nullable, then:
   `alembic revision --autogenerate -m "add google_sub to users"`
   Inspect it before applying. Autogenerate is unreliable about nullability
   changes, so confirm the `alter_column` for password_hash is actually present
   and that downgrade -1 reverses both changes.
3. `app/config.py`: the three settings above.
4. `app/services/google_auth.py`:
   - `build_authorization_url(state)` returning the URL to redirect to
   - `exchange_code(code)` returning a small dataclass of sub, email,
     email_verified, and name, raising on a bad or expired code
   - `find_or_create_user(db, identity)` implementing the three step resolution
     above, raising a distinct error for the unverified-email refusal
   Keep the HTTP calls in this module. Nothing else imports authlib.
5. `app/routers/auth.py`, two routes:
   - `GET /auth/google/start`: generate a random state token, set it in a short
     lived httpOnly cookie (10 minutes, SameSite=Lax, same secure flag as the
     session cookie), 302 to Google.
   - `GET /auth/google/callback`: compare the returned state against the cookie
     and reject a mismatch. Exchange the code, resolve the user, create a
     session, set the session cookie, clear the state cookie, then 302 to
     `settings.site_url`.

   Both return redirects, not JSON. The browser arrives here from Google, not
   from `apiFetch`. On any failure redirect to `{site_url}/login?error=<code>`
   rather than rendering a bare 400, so the user lands somewhere usable.
6. `tests/test_google_auth.py`, with `exchange_code` monkeypatched so no test
   touches the network:
   - a new Google identity creates a user with password_hash None and sets a
     session cookie
   - signing in twice with the same sub reuses the account rather than
     duplicating it
   - a Google identity whose email matches an existing password account links to
     it when email_verified is true
   - the same case with email_verified false is refused and does not link
   - a state mismatch is rejected
   - `POST /auth/login` with a password against a Google-only account returns
     401, not 500
   - the existing leak test still passes

## Frontend tasks
1. No addition to `src/api/auth.js`. This is a full page redirect, not a fetch,
   so it must not go through `apiFetch`.
2. `src/components/GoogleButton/` with its own CSS Module: a link to
   `${import.meta.env.VITE_API_URL}/api/v1/auth/google/start`, styled to
   Google's branding guidelines, with a divider reading "or" separating it from
   the email form. Place it on both Login and Register.
3. `AuthContext` already calls `getMe()` on mount, so the redirect back from the
   callback picks up the new session with no change. Verify this rather than
   assuming it.
4. Login page: read `?error=` from the query string and render a readable
   message. At minimum handle the unverified-email refusal, which needs to
   explain that the account should be signed into with its password instead.

## A cookie note that will look wrong and is not
In production the callback runs on `api.abacadaba.com` and the frontend is
`abacadaba.com`, which works because `SESSION_COOKIE_DOMAIN=.abacadaba.com`
already covers both.

Locally there is no shared parent domain, but cookies ignore port numbers. A
cookie set by `localhost:8000` with no domain attribute is sent to
`localhost:5173`. The local configuration needs no change. Do not "fix" this by
setting a domain locally.

## Acceptance criteria
- alembic upgrade head adds google_sub and makes password_hash nullable,
  downgrade -1 reverses both
- clicking the button at localhost:5173/login reaches Google's consent screen
- consenting returns to the app already signed in, with the header showing the
  Google account's name
- that account appears in the database with password_hash NULL
- signing out and back in through Google reuses the same account, and
  `select count(*) from users` does not grow
- an existing password account signing in through Google with the same verified
  email links rather than erroring or duplicating
- attempting to sign in with a password to a Google-only account returns 401
- tampering with the state parameter is rejected
- an existing password account can still register, sign in, and sign out exactly
  as before
- pytest passes, including the leak test

## When done
Append an entry to CHANGELOG.md and stop.
