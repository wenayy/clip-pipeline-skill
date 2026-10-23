# Clip Selection — deep dive

How `analyzer.pick_clips` finds the best moments from the whole file. Read this
when changing scoring, windowing, ranking, or boundary refinement. All code is
in `backend/app/pipeline/analyzer.py` unless noted.

## Why not just send the whole transcript to the LLM?

For short videos it does (`if total <= WINDOW_SIZE`). For long ones, one giant
prompt is worse, not better:

- **Context limit** — a 2-hour transcript with word timings can exceed the model
  window.
- **"Lost in the middle"** — models read long inputs carelessly, attending to the
  start/end and skimming the middle. Short windows get read closely. This is the
  main reason, not just token cost.
- **Bigger candidate pool** — windowing yields ~6×(#windows) candidates to rank,
  vs one skim of everything.
- **Fault tolerance + progress** — small parallel-able calls; one bad window
  doesn't sink the job; the UI can show "reviewing moment N of M".

The whole-video view is recovered *afterward* by pooling candidates and
comparing finalists, so you get close-reading first, global comparison second.

## The signals

1. **Acoustic (`energy.py`)** — per utterance: `loudness` (RMS normalized against
   the whole video) and `dynamics` (how much loudness swings within a line; flat
   = monotone). Attached to each utterance and surfaced to the LLM via
   `energy.summarize`. Not a decider on its own — a loud argument about parking
   isn't clippable — so it's a feature fed into the rubric. Degrades to a no-op
   on any failure.

2. **Rubric (the LLM)** — each candidate scored 1–10 on six axes. The prompt
   *defines* each axis so the model applies our definition, not a vague instinct:
   - `hook` — does the first line open an unresolved question (stop the scroll)
   - `standalone` — understandable with zero context (detects dangling pronouns /
     back-references like "he", "that thing we said")
   - `emotion` — real charge: funny, shocking, moving
   - `ending` — lands on a strong line, not mid-thought
   - `payoff` — the hook's promise is delivered
   - `share` — would someone send it to a friend

## Scoring math

`WEIGHTS = {hook:0.26, standalone:0.20, emotion:0.18, ending:0.16, payoff:0.12,
share:0.08}`. Final score = weighted mean: `sum(WEIGHTS[k] * scores[k])`. One
number per candidate → directly comparable. Missing axis defaults to 5.

`scoring.py` runs the raw score through a **monotonic** display curve
(`presentation_score`) so a top-of-hundreds finalist doesn't show as "6.9". The
curve changes the *displayed* number only — ranking uses the raw score, so order
is unchanged.

## The selection algorithm — three passes (the code labels them this way)

**Pass 1 — nominate per window** (`_collect_candidates`):
1. **Window** (`_windows`) — `[start, start+110)` ranges stepping by
   `WINDOW_SIZE - WINDOW_OVERLAP = 90`, covering all utterances with 20 overlap.
2. Each window sends a numbered slice (`_numbered_slice`: `[i] (start-end)
   <energy hint><text>`) and returns up to `CANDIDATES_PER_WINDOW = 6`
   candidates as `{start_index, end_index, title, scores}`.

**Pass 2 — comparative selection** (head-to-head, `analyzer.py` ~line 442):
3. Pool all candidates, sorted by weighted score. Per-window scores are NOT
   calibrated against each other (a weak window's 7 is not a strong window's 7),
   so the finalists go through a direct comparison (prompt: "Pick the N
   strongest… **Compare directly**: if two [compete], keep the better one"). A
   floor guarantees a minimum number of finalists reach this pass.
4. **Greedy de-overlap** (`_overlaps`) — walk best→worst, keep a clip only if it
   doesn't overlap one already chosen; the duplicate from window overlap
   collapses to the better one.

**Pass 3 — boundary refinement** (`_refine_boundaries`, ~line 584):
5. Snap the chosen ranges to clean sentence-level cuts, word-accurate. Then
   `resolve_clip_times` turns indices → real seconds from word timestamps and
   collects the clip's `words`; for a refined clip `_trim_weak_opening` and
   `_strip_trailing_junk` shave filler openers ("um, so…") and trailing junk.

(Higher tiers run all three passes; the mode constant near the top of
`analyzer.py` controls whether the cheaper pass-1-only path is used.)

## Why index ranges, not timestamps

The LLM only counts line numbers in what it was shown — reliable. It is bad at
inventing floating-point seconds — unreliable. So it answers "clip = utterances
91→96" and Python looks up `utterances[91]["start"]` … `utterances[96]["end"]`.
This is also why utterances are presented as a numbered, in-order script: the
model reads the flow (so it understands structure — setup → payoff) and the
numbers are just the addressing scheme for its answer.

## What to preserve when editing

- The **index-range contract** between LLM and `resolve_clip_times`.
- **Overlap** ≥ the longest single clip's utterance span, so no clip is split.
- The **pool → global rank → de-overlap** shape (don't let a single window's
  absolute scores decide the final set).
- The six axes and their **weights** — they encode the product's taste. Change
  them deliberately, not incidentally.
