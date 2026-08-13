# Current Feature

## Feature 005, Taking the quiz with per answer confetti

## Goal
The user answers one question at a time. The server grades each answer. A
correct answer fires a small burst of confetti. The quiz advances until all five
are answered, then shows a plain "quiz complete" placeholder.

## In scope
- A grading endpoint that checks a single answer
- One question at a time navigation with selection state
- Small confetti burst on a correct answer
- Correct and incorrect visual feedback, including revealing the right choice
- A minimal end of quiz placeholder
- Tests for grading and for tampering

## Out of scope
- Scoring, pass or fail at 4/5, the attempts table, big confetti (feature 006)
- Certificates, auth, retaking, resuming a partly finished quiz
- Persisting anything about this session. Answers live in React state only.

## A known limitation, deliberately accepted
Because grading is one request per answer with no server side session, someone
could call the endpoint repeatedly to discover the correct choice. That is fine
for now. Feature 006 introduces the attempts table, which records answers server
side and makes an attempt single submission. Add a brief comment saying so in
the grading router. Do not try to solve it in this feature.

## Backend tasks
1. app/schemas/quiz.py, add:
   - AnswerRequest: question_id int, choice_id int
   - AnswerResponse: correct bool, correct_choice_id int
   correct_choice_id is only ever returned after the answer has been graded, so
   it does not violate the leak rule. ChoicePublic still must not gain
   is_correct.
2. app/services/quiz.py, add grade_answer(db, slug, question_id, choice_id):
   - Load the question, confirming it belongs to a published lesson with that
     slug. Return a not found signal otherwise.
   - Confirm choice_id belongs to that question. If it does not, that is a bad
     request, not a wrong answer.
   - Return whether the chosen choice is correct, plus the id of the correct one.
3. app/routers/quiz.py: POST /lessons/{slug}/quiz/answers
   - 404 when the lesson or question does not exist or the lesson is unpublished
   - 400 with a clear detail when the choice does not belong to the question
   - 200 with AnswerResponse otherwise
4. tests/test_quiz.py, add:
   - a correct choice returns correct true with the matching correct_choice_id
   - an incorrect choice returns correct false and still reveals
     correct_choice_id
   - a choice_id belonging to a different question returns 400
   - a question_id from a different lesson returns 404
   - an unknown question_id returns 404

## Frontend tasks
1. npm install canvas-confetti. This is the one new dependency. It is small,
   framework agnostic, and draws to its own canvas.
2. src/api/quiz.js: add submitAnswer(slug, questionId, choiceId).
3. src/lib/confetti.js: wrap canvas-confetti in two named functions,
   smallBurst() and bigBurst(). Implement smallBurst now as a modest burst of
   roughly 40 particles originating near the answered choice or the center of
   the card. Define bigBurst as a stub with a comment pointing at feature 006 so
   the module is ready. Respect prefers-reduced-motion: if the user has it set,
   both functions do nothing.
4. src/pages/Quiz/: convert from the read only list to one question at a time.
   State to track: current question index, the selected choice for the current
   question, whether the current answer has been graded, the grading result, and
   an array of results so far.
   Flow:
   - Show question N of 5 with a progress indicator
   - The user clicks a choice. It highlights as selected. Nothing is graded yet.
   - A Submit button becomes enabled. Clicking it calls the API.
   - While the request is in flight, disable the choices and the button.
   - On a correct result, mark the chosen choice correct and fire smallBurst().
   - On an incorrect result, mark the chosen choice incorrect and also mark the
     choice matching correct_choice_id as the right answer.
   - After grading, the choices stay locked and a Next button appears.
   - Next advances to the following question with fresh state. On the last
     question the button reads Finish.
   - Finish shows a placeholder panel reading "Quiz complete" with a count of
     how many were answered, and a note that scoring and certificates arrive
     next. Include a link back to the lesson.
   Keep the page component under about 150 lines. Push the per question view
   into QuestionCard and extract a small ProgressBar component if needed.
5. src/components/QuestionCard/: accept the choices, a selected id, a graded
   result, and callbacks. Four visual states per choice: default, selected,
   correct, and incorrect. Use CSS Module classes driven by props, with colors
   from global.css custom properties. Add new properties there for the correct
   and incorrect colors rather than hardcoding.
6. Accessibility: choices are real buttons, reachable by keyboard, with a
   visible focus ring. When a result comes back, announce it in an aria-live
   polite region so it is not conveyed by color alone.
7. Handle a failed grading request without losing the user's place: show an
   inline error with a Retry button that resubmits the same answer.

## Acceptance criteria
- curl a correct answer to the grading endpoint and it returns correct true
- curl a wrong answer and it returns correct false with correct_choice_id
- curl a choice_id from a different question and it returns 400
- at localhost:5173 the quiz shows one question at a time with a progress
  indicator
- selecting a choice highlights it, and Submit grades it
- a correct answer fires a small confetti burst
- a wrong answer marks the choice red and reveals the correct one in green
- choices lock after grading and Next advances
- finishing all five shows the "Quiz complete" placeholder
- the whole quiz is completable using only the keyboard
- pytest passes, including the existing leak test

## When done
Append an entry to CHANGELOG.md and stop.
