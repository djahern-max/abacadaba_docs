# Current Feature

## Feature 007, Certificate generation and download

## Goal
A passed attempt earns a downloadable PDF certificate carrying the recipient's
name, the lesson title, the score, the date, and a verification code that
resolves on a public page.

## In scope
- recipient_name and certificate_code columns on attempts, with a migration
- A name entry step on the results page for a passing attempt
- Server side PDF generation
- A download endpoint and a public verification page
- Tests

## Out of scope
- Auth and accounts (feature 008). The name is typed by the user.
- Emailing the certificate, storing PDFs in Spaces, certificate templates or
  themes, revocation, expiry
- Certificates for failed attempts

## Honest framing of what the certificate proves
With no accounts, the name is self reported. The certificate and the
verification page therefore assert that an attempt with this code passed this
lesson on this date with this score, and that the name was supplied by the
person taking it. Word the verification page accordingly rather than implying
verified identity. Add a short comment in the certificate service saying feature
008 replaces the typed name with an authenticated user.

## Data model
Add to attempts:
- recipient_name: string, nullable. Set when the user claims the certificate.
- certificate_code: string, unique, indexed, nullable. Generated when the
  certificate is first claimed, not at attempt creation.

Generate the code with secrets.token_urlsafe trimmed to a readable length,
uppercased, and hyphenated into groups, for example ABCD-EFGH-JKMN. Exclude
characters that are easy to confuse: no O, 0, I, 1, or L. Put the alphabet in a
module level constant.

## Backend tasks
1. Add reportlab to requirements.txt and install it. It is pure Python with no
   system dependencies, which keeps deployment simple. No other new dependency.
2. Add both columns to the Attempt model, then:
   alembic revision --autogenerate -m "add certificate fields to attempts"
   alembic upgrade head, and verify downgrade -1 reverses.
3. app/schemas/certificate.py:
   - CertificateClaim: recipient_name, a string of 2 to 80 characters after
     stripping whitespace. Reject an empty or whitespace only name.
   - CertificateInfo: certificate_code, recipient_name, lesson_title, score,
     question_count, completed_at
   - CertificateVerification: valid bool, plus the same fields when valid
4. app/services/certificates.py:
   - generate_code() using the confusion free alphabet, retrying on the
     astronomically unlikely collision.
   - claim_certificate(db, public_id, recipient_name) which loads the attempt,
     refuses unless it is complete and passed, sets recipient_name, generates
     certificate_code only if one is not already set, commits, and returns
     CertificateInfo. Claiming twice with a different name updates the name but
     keeps the original code.
   - render_pdf(info) returning PDF bytes via reportlab, drawn on a landscape
     A4 or letter page. Include: a heading reading Certificate of Completion,
     the recipient name as the visual focus, the lesson title, the score written
     as "Scored 4 out of 5", the completion date formatted readably, the
     abacadaba wordmark, and the verification code in small text at the foot
     alongside the verification URL. Center the layout and handle a long name or
     lesson title by shrinking the font rather than overflowing the page.
   - verify_code(db, code) returning verification data or a not found signal.
     Match case insensitively and tolerate missing hyphens.
5. app/routers/certificates.py:
   - POST /attempts/{attempt_id}/certificate with CertificateClaim.
     200 with CertificateInfo, 404 unknown attempt, 409 if the attempt is
     incomplete or did not pass, 422 on an invalid name.
   - GET /attempts/{attempt_id}/certificate.pdf returning a StreamingResponse
     of application/pdf with a Content-Disposition attachment filename built
     from the lesson slug, for example abacadaba-intro-to-ratios.pdf.
     404 unknown attempt, 409 if not yet claimed or not passed.
   - GET /certificates/{code} returning CertificateVerification. Return 200 with
     valid false for an unknown code rather than a 404, so the public page can
     render a clean "not found" state.
   Register under /api/v1.
6. tests/test_certificates.py:
   - claiming on a passed attempt returns a code and the name
   - claiming on a failed attempt returns 409
   - claiming on an incomplete attempt returns 409
   - claiming twice keeps the original code and updates the name
   - an empty or whitespace only name returns 422
   - the PDF endpoint returns 200, content type application/pdf, and bytes
     starting with %PDF
   - the PDF endpoint before claiming returns 409
   - verifying a real code returns valid true with the right lesson and score
   - verifying an unknown code returns 200 with valid false
   - verifying is case insensitive and tolerates missing hyphens

## Frontend tasks
1. src/api/certificates.js: claimCertificate(attemptId, name),
   certificatePdfUrl(attemptId) returning the URL string, and
   verifyCertificate(code).
2. src/pages/Result/: replace the disabled placeholder button. On a pass, show a
   name input labeled "Name as it should appear on your certificate" and a
   "Get my certificate" button. On submit, claim the certificate, then show the
   code and a "Download PDF" link pointing at the PDF URL. Disable the button
   while the request is in flight, and show inline validation for an empty name
   rather than relying on the server 422 alone. If a certificate is already
   claimed for this attempt, show the code and download link straight away
   instead of the form.
3. src/pages/Verify/: route "/verify/:code". Fetch on mount and render either
   the certificate details with wording that reflects the self reported name, or
   a clear not found state. This page must be readable by someone with no
   context, so include the lesson title, score, and date plainly.
4. Register the route in App.jsx. Make sure the verification URL printed on the
   PDF matches this route. Read the site base URL from an env var,
   VITE_SITE_URL on the frontend and SITE_URL on the backend, defaulting to
   http://localhost:5173. Add both to the .env.example files.
5. The download link should be a plain anchor with the download attribute
   pointing at the API URL, not a fetch and blob dance.

## Acceptance criteria
- alembic upgrade head adds both columns, downgrade -1 reverses them
- curl claiming a certificate on a passing attempt returns a code
- curl claiming on a failing attempt returns 409
- curl -o test.pdf on the PDF endpoint produces a file that opens in Preview and
  shows the name, lesson, score, date, and code
- a long name does not overflow the page
- curl on /api/v1/certificates/{code} returns valid true, and lowercase and
  unhyphenated variants of the code also work
- an unknown code returns valid false, not a 500
- passing a quiz in the browser, entering a name, and clicking through downloads
  a real PDF
- visiting /verify/{code} in the browser shows the certificate details
- pytest passes, including the leak and replay tests

## When done
Append an entry to CHANGELOG.md and stop.
