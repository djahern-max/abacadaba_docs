019 — Courses as the credit-bearing unit

The one to do first, and it's structural. In the Standards, credit attaches to a program, not to a video. Your lesson is a segment; a 1-credit course is several segments plus an assessment. Right now attempts and certificates hang off lessons, which means a lesson is implicitly the credit unit — and that assumption is baked into six features already.

Introduce courses, make lessons ordered members of a course, and move attempts and certificates up to the course. Painful migration, zero rows to migrate today. Every feature below depends on this one existing, and every day you delay it the migration gets worse.

020 — Program metadata and learning objectives

Learning objectives as ordered rows on a course (measurable verbs, validated non-empty at publish), plus program level, prerequisites, advance preparation, and field of study as an enum populated from the Fields of Study PDF. These must be disclosed to participants before they register, so they render on the course detail page, not just in the admin editor.

021 — Development and review chain

The SME gate. developer_id, reviewer_id, reviewed_at, review_notes on a course, plus a sources table (citation, URL, retrieval date) linked to it. Publish is blocked when there's no reviewer and blocked when reviewer equals developer.

Build this before the LLM pipeline, not after. If the pipeline exists first, you'll design it around one person clicking through, and the second signature becomes a retrofit.

022 — Credit measurement

Store the inputs, not just the answer: word count, A/V minutes, question count, the computed credit, and the formula version used. Show the arithmetic in the admin UI. Recompute when any input changes and refuse to publish with stale numbers.

Nice reuse here — feature 017's auto-filled video duration becomes the A/V input directly. And build in the narration rule: a flag on each segment for whether its audio constitutes additional learning or narration of the text, because that flag decides whether you count words or runtime.

023 — Review questions, assessment questions, and thresholds

Your app has one kind of question. The Standards have two, with different rules. Review questions are formative and scored for feedback; the final assessment is what gates credit. Split the type, enforce the floors against the course's computed credit (at least 3 review and 5 assessment questions per credit, no true/false or yes/no items), and make the pass threshold a column rather than PASS_THRESHOLD = 4.

This is where your existing five-questions-per-lesson validation rule finally has to become credit-derived instead of hardcoded.

024 — Completion documents and participant records

Certificates carrying every required field: sponsor name and ID, participant name, course title, field of study, delivery method, credits, completion date, program level. Then the records side — an admin view of completions with export, because sponsors must produce these on audit.

025 — Evaluations

Post-completion survey with the required dimensions, and an admin view of the results. Small feature, mandatory, and it gives you data on whether your AI-drafted content is any good.

026 — Policies, disclosures, and content currency

Refund and cancellation, complaint resolution, records retention statements as real pages. Plus last_reviewed_at on courses and a dashboard of what's overdue, since materials must stay current.