# Current Feature

## Feature 009, Deploy to production

## Goal
abacadaba runs on the public internet at a real domain with TLS, a managed
Postgres, and automatic deploys from main. Everything that works locally works
there, including session cookies and video playback.

## In scope
- Backend containerized and deployed to DigitalOcean App Platform
- Frontend built and served as a static site on the same App
- Managed Postgres, with migrations run automatically on release
- Production environment configuration, including secure cookies
- Domain and TLS for abacadaba.com and api.abacadaba.com
- A production seed run, and the first admin account

## Out of scope
- CDN tuning, video transcoding, adaptive streaming
- Staging environments, blue green deploys, rollback automation
- Monitoring and alerting beyond the platform's built in health check
  (feature 013)
- Backups configuration beyond confirming the managed default exists

## The decision that prevents a day of cookie debugging
Put the API on api.abacadaba.com and the frontend on abacadaba.com. These share
a registrable domain, so the session cookie is same site and SameSite=Lax works
unchanged. If the API lived on a different domain the cookie would need
SameSite=None with Secure, and Safari's tracking prevention would make it
unreliable. Do not put them on different domains.

Set the cookie domain to .abacadaba.com in production so it is sent to both
hosts. Locally it stays host only. This comes from config, not a hardcoded
string.

## Backend tasks
1. backend/Dockerfile: python:3.12-slim base, non root user, install from
   requirements.txt, copy the app, expose 8080, and run uvicorn bound to
   0.0.0.0 on the port from the PORT env var defaulting to 8080. Multi stage is
   unnecessary here, keep it one stage and readable.
2. backend/.dockerignore excluding .venv, __pycache__, tests, .env, alembic
   caches.
3. app/config.py: add SESSION_COOKIE_DOMAIN (nullable), ENVIRONMENT defaulting
   to "development", and confirm SESSION_COOKIE_SECURE, SITE_URL, and
   CORS_ORIGINS are all present. Add every new var to .env.example.
4. Cookie setting code reads domain and secure from settings rather than
   literals. Verify SameSite is Lax.
5. app/main.py: when ENVIRONMENT is production, disable the interactive docs by
   passing docs_url=None and redoc_url=None. The API is public, the schema does
   not need to be.
6. A release command that runs alembic upgrade head before the new version takes
   traffic. On App Platform this is the job or pre deploy command, not something
   baked into the container start, so a crash loop cannot half apply a migration.
7. Confirm the existing /api/v1/health endpoint is what the platform health
   check points at.
8. backend/scripts/seed.py must be safe to run against production. Re read it and
   confirm it only ever upserts and never deletes.

## Frontend tasks
1. frontend/.env.production with VITE_API_URL=https://api.abacadaba.com and
   VITE_SITE_URL=https://abacadaba.com. Confirm nothing reads a localhost value
   at build time.
2. Client side routing means a refresh on /lessons/foo must not 404. Configure
   the static site to rewrite unknown paths to index.html. Note the exact
   setting used in the deployment doc below.
3. npm run build must succeed with no warnings about missing env vars.

## Deployment tasks
1. Create the App with two components pointed at this GitHub repo: a service
   from backend/ using its Dockerfile, and a static site from frontend/ built
   with npm run build and served from dist/.
2. Attach a managed Postgres. Use the platform's connection string binding so
   DATABASE_URL is injected rather than pasted. It must connect over the private
   network.
3. Set all secrets as encrypted env vars: SPACES_KEY, SPACES_SECRET, and the
   rest. Nothing sensitive goes in a build arg, since those are visible.
4. Point abacadaba.com at the static site and api.abacadaba.com at the service.
   Let the platform issue certificates.
5. Enable automatic deploys from main.
6. After the first successful deploy, run the seed once and create the first
   admin with scripts/make_admin.py through the platform console.
7. Upload one real video to the production lesson so the deployed app has
   working content.

## Write it down
Create DEPLOYMENT.md at the repo root recording: the App components and their
source directories, every environment variable and where its value comes from,
the SPA rewrite setting, how to run a one off command in production, how to get
a shell on the database, and how to promote an admin. Future you will need this
and will not remember.

## Acceptance criteria
- https://abacadaba.com loads over TLS with a valid certificate
- the lesson list renders from the production database
- a video plays, proving Spaces credentials and presigned URLs work in production
- refreshing on a deep link like /lessons/intro-to-ratios does not 404
- registering an account on production works and the session cookie has Secure,
  HttpOnly, SameSite=Lax, and the .abacadaba.com domain
- signing out and back in works, proving the cookie survives across both hosts
- completing a quiz produces a certificate whose verification URL points at the
  real domain, not localhost
- https://api.abacadaba.com/docs returns 404 in production
- pushing a commit to main triggers a deploy that runs migrations first
- DEPLOYMENT.md exists and is accurate

## When done
Append an entry to CHANGELOG.md and stop.
