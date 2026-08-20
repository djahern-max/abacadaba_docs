# Current Feature

## Video pipeline 01, Narration generation with measured reveals

## Goal
`npm run generate` produces narration audio for a lesson, measures its real
duration, and derives reveal timings from the audio itself rather than from
hand-written guesses. After this feature, no timing number in the video package
is estimated once audio exists.

## Numbering note, read this first
This is not feature 022. Feature 022 is credit measurement in the app, and it
lives in `backend/` and `frontend/`. This work is in `video/`, which is a build
tool, not application code, and nothing in the app imports from it.

The two are related in one direction only: this feature produces the accurate
A/V runtime that 022 will eventually take as an input. It does not compute or
store credit, and it must not start doing so.

If the `current-feature_NN.md` numbering is meant to describe app features, give
the video package its own track rather than consuming a number from the app's
sequence. That is the assumption this file makes.

## In scope
- Wiring `scripts/generate-audio.ts` into the package
- Replacing `durations.json` with `audio-meta.json`, which carries duration,
  measured reveals, and a content hash per block
- Reveal markers in narration text, and the helpers that strip them
- Retiring `scripts/measure-audio.mjs` and the manual `AUDIO_PRESENT` map
- Marking up block-01 only, as a proof of the approach

## Out of scope
- Marking up blocks 02 through 07. Deliberate: block-01 gets listened to and
  judged before spending marker effort on six more sheets.
- Pronunciation dictionaries. Nothing gets written until we have heard which
  words are actually wrong.
- Any change to slide components in `slides.tsx`.
- Anything in `backend/` or `frontend/`.

## Read this before starting

**There is a misplaced file.** `audio-meta.json` was created at
`frontend/src/audio-meta.json`. It does not belong there and `frontend/` must
not be touched by this feature. Delete it and create the file at
`video/src/audio-meta.json` instead. Confirm `frontend/` is clean afterward.

**`lesson-01.ts` is currently in a broken intermediate state.** It imports
`audioMeta` at the top but the bottom of the file still references `measured`,
which was the old `durations.json` import and no longer exists. Do not try to
preserve both. The bottom of the file is being replaced wholesale.

**Estimated durations must never reach a credit calculation.** This is the
constraint the whole package is shaped around, under Standards paragraph 7.02.7.
`usingEstimates` is what enforces it, and it currently has a bug: it counts the
title sheet, which has no narration and will never have audio, so it can never
become false. Fix it as specified below. Do not remove the warning in `Root.tsx`.

## Reveal markers
Narration text may contain `[[r]]` markers. Each marks a point where a slide
element should appear. Markers are stripped before the text is sent to
ElevenLabs; their positions are then located in the returned character-level
alignment data to produce exact seconds.

The number of markers in a block must equal the length of that block's
`reveals` array, because the slide components index into it positionally. If
they disagree, the slide will read `undefined` and elements will never appear.

`narration` remains the transcript of record. Markers live inside it, and
`transcriptOf()` strips them. There is deliberately no second copy of the
narration text to drift out of sync.

## Tasks

### 1. Fix the misplaced file
- Delete `frontend/src/audio-meta.json`
- Create `video/src/audio-meta.json` containing exactly `{}`
- Verify `git status` shows no changes under `frontend/`

### 2. `video/package.json`
- Add to scripts: `"generate": "tsx scripts/generate-audio.ts"`
- Remove the `"measure"` script
- `tsx` is already in devDependencies; leave it

### 3. `video/scripts/generate-audio.ts`
Already present. Do not modify it. It is the reference for what the rest of the
package must expose — if something does not compile, the fix goes in
`lesson-01.ts`, not here.

### 4. `video/src/lesson-01.ts`

Update the file-level doc comment: the duration resolution order is now
`audio-meta.json` first, `estimatedSeconds` second. It currently names
`durations.json` and `npm run measure`, both of which are being retired.

In the `Block` type, restore the misplaced comment and add the optional field:

```ts
  narration: string;      // transcript of record, may contain [[r]] markers
  reveals: number[];      // fallback seconds from block start, used until measured
  speech?: string;        // overrides narration for TTS only; rarely needed
```

Replace everything from `const measuredMap` to the end of the file with:

```ts
type BlockMeta = { durationSeconds: number; reveals: number[]; hash: string };
const audio = audioMeta as Record<string, BlockMeta>;

/** The transcript of record: markers stripped, nothing else changed. */
export const transcriptOf = (b: Block): string =>
  b.narration.replace(/\s*\[\[r\]\]\s*/g, " ").replace(/\s+/g, " ").trim();

/** What gets sent to ElevenLabs. Markers intact; the script strips them. */
export const speechOf = (b: Block): string => b.speech ?? b.narration;

export const hasAudio = (b: Block): boolean => audio[b.id] !== undefined;

export const durationOf = (b: Block): number =>
  audio[b.id]?.durationSeconds ?? b.estimatedSeconds;

/** Measured reveals when we have them, hand-written estimates when we do not. */
export const revealsOf = (b: Block): number[] =>
  audio[b.id]?.reveals ?? b.reveals;

/**
 * Blocks with empty narration have no audio by design — the title sheet is the
 * only one. Counting it here would make this permanently true and the warning
 * in Root.tsx permanently useless.
 */
export const usingEstimates = blocks.some(
  (b) => b.narration.trim().length > 0 && !hasAudio(b)
);

export const totalSeconds = blocks.reduce((sum, b) => sum + durationOf(b), 0);
```

Then add markers to block-01's narration, and only block-01. Three markers, to
match its three-entry `reveals` array. Insert them exactly here, changing no
other character of the text:

- Before `you have said the phrase percentage of completion` — the eyebrow and
  headline appear as the narrator names the phrase.
- Before `method. It stopped being one.` — note this is mid-sentence, before
  the word `method` rather than at the start of the sentence. The visual is a
  strikethrough drawing across that specific word, so it fires as the word is
  spoken.
- Before `What replaced it is a sequence of separate judgments` — the third
  element.

Leave `reveals: [1, 14, 26]` in place. It is the fallback until audio exists.

### 5. `video/src/Lesson.tsx`
- Import `revealsOf` and `hasAudio` from `./lesson-01`
- Delete the `AUDIO_PRESENT` constant entirely, and its comment
- `<Slide reveals={revealsOf(block)} />`
- `{hasAudio(block) ? <Audio src={staticFile(...)} /> : null}`

`AUDIO_PRESENT` was a hand-maintained map that had to be pasted in after every
generation run. `audio-meta.json` already knows, so the manual step goes away.
Do not replace it with anything.

### 6. Retire the old measurement path
- Delete `video/scripts/measure-audio.mjs`
- Delete `video/src/durations.json`
- Confirm nothing imports either. `Root.tsx` imports only `durationOf`,
  `totalSeconds`, and `usingEstimates`, all of which still exist.

### 7. `video/tsconfig.json`
Add `scripts` to `include` if it is not already covered, so the editor does not
report phantom errors in `generate-audio.ts`. `resolveJsonModule` must stay on;
`audio-meta.json` depends on it exactly as `durations.json` did.

### 8. `video/README.md`
The build order section describes a four-step manual flow that no longer exists.
Rewrite steps 2 and 3 as one step: `npm run generate`. Keep the two compliance
notes at the bottom unchanged — the estimated-durations rule and the
transcript-is-a-supplement rule are both still true and both still matter.

Document the marker convention in the structure section.

## Acceptance
Run these in order. Each must pass before moving on.

1. `npx tsc --noEmit` — clean. No reference to `measured` or `durations.json`
   survives anywhere.
2. `npm run generate -- --dry-run` — lists seven blocks, `block-01` through
   `block-07`. The title sheet is absent, which is correct. `block-01` reports
   3 reveals; every other block reports 0.
3. `npm run dev` — Remotion Studio opens, the composition is its full estimated
   length, all eight sheets render, and the console shows the estimated-duration
   warning from `Root.tsx`.

Do not run `npm run generate` for real. That spends API credits and is the
human's step, not yours.

## Do not
- Touch `backend/` or `frontend/`
- Modify `slides.tsx`
- Add markers to blocks 02 through 07
- Compute, store, or print anything described as a course credit. The sanity
  check already in `generate-audio.ts` is labelled as such and is the only
  credit arithmetic permitted in this package.
- Commit `.env`. Verify it is still ignored before finishing.
