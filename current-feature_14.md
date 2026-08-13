# Current Feature

## Feature 014, Header cleanup and password field polish

## Goal
The header stops advertising a debugging aid to end users, and the password
fields behave the way people expect: typed twice when creating an account, and
readable on demand.

## In scope
- Removing the backend status pill from the header
- A confirm password field on Register
- A show/hide toggle on every password field
- Tests only where tests already exist

## Out of scope
- Password reset, email verification, password strength meters. All worth doing,
  none of them this.
- Changing the 10 character minimum or adding complexity rules
- Any backend change at all. The API contract does not move.
- Adding a frontend test framework. There isn't one, and deciding to introduce
  Vitest is its own conversation, not a rider on a cleanup feature.

## Read this before starting
This is a small feature. If it grows a migration, a new endpoint, or a new
dependency, something has gone wrong.

## Part 1, remove the status pill
The pill was feature 001's proof that the walking skeleton was wired up. It has
done its job. It currently tells every visitor "Backend connected," which means
nothing to them and looks like a leftover.

Remove:
- The three pill branches in `components/Header/Header.jsx`
- The `status` prop from Header, and the `useState`/`useEffect`/`getHealth`
  block in `App.jsx` that feeds it
- The `.pill`, `.connected`, `.unreachable`, and `.checking` rules in
  `Header.module.css`
- `src/api/health.js`, but only after grepping to confirm nothing else imports
  `getHealth`

Do **not** remove:
- `GET /api/v1/health` or `app/routers/health.py`. Production health checking
  points at it, and it is the DB-backed check DEPLOYMENT.md explains at length.
  This feature deletes a UI element, not an endpoint.
- The `--color-success-bg` / `--color-danger-bg` custom properties in
  `global.css`. The pill classes used them, but so does the admin stats page.

After this, the header is the wordmark on the left and the auth nav on the
right. Check both signed-in and signed-out states for spacing that only looked
right because the pill was filling the gap.

## Part 2, a PasswordInput component
Both pages need the same toggle, so build it once:
`src/components/PasswordInput/PasswordInput.jsx` plus its CSS Module.

Props: `id`, `label`, `value`, `onChange`, `disabled`, `autoComplete`, and an
optional `hint` rendered under the field.

It renders the label, the input, and a toggle button positioned inside the
field. Internal `useState` holds whether the text is visible; the input's `type`
flips between `password` and `text`.

Requirements that are easy to get wrong:
- The toggle is `<button type="button">`. The default type inside a form is
  `submit`, so omitting this makes the eye icon submit the registration form.
  This is the single most likely bug in this feature.
- Give it an `aria-label` that changes with state: "Show password" when hidden,
  "Hide password" when visible. An unlabelled icon button is invisible to a
  screen reader.
- Inline SVG for the icon. No icon library — that would be a new dependency for
  two glyphs.
- Visible focus ring, consistent with the quiz choice buttons from feature 005.
- Visibility state is per field and resets on navigation. Do not persist it.

## Part 3, wire it into the forms
`Login.jsx`: replace the password input with `PasswordInput`,
`autoComplete="current-password"`. No confirm field here — re-typing a password
to sign in serves no purpose and only adds friction.

`Register.jsx`: replace the password input with `PasswordInput`
(`autoComplete="new-password"`), and add a second one below it labelled
"Confirm password", also `autoComplete="new-password"`, backed by a new
`confirmPassword` state variable.

Extend the existing `validate()` rather than adding a second validation path.
Order the checks name, then length, then match, so someone who types a short
password twice is told the useful thing first:

```js
if (password !== confirmPassword) {
  return 'Passwords do not match.'
}
```

Validate on submit, matching how the form already behaves. Do not validate on
every keystroke — flashing "do not match" while someone is still typing the
second field is worse than saying nothing.

Leave the "At least 10 characters." hint under the first password field only.

## What must not regress
The Google button and the `or` divider are new as of feature 013 and sit on both
pages. Confirm the layout still holds with an extra field above them on Register,
and that a browser password manager still autofills the sign-in form — the
`autoComplete` values above are what make that work.

## Acceptance criteria
- the header shows only the wordmark and the auth nav, signed in and signed out
- `curl http://localhost:8000/api/v1/health` still returns 200 with the DB check
- registering with mismatched passwords shows "Passwords do not match." and does
  not call the API
- registering with matching passwords still succeeds and signs the user in
- the eye toggle reveals and hides on all three fields
- clicking the eye on Register does not submit the form
- every field and its toggle are reachable and operable by keyboard alone
- a password manager still offers to fill the sign-in form
- signing in with Google still works from both pages
- `npm run lint` passes
- pytest passes, unchanged, confirming nothing backend moved

## When done
Append an entry to CHANGELOG.md and stop.
