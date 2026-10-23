---
name: clip-pipeline
description: >-
  The full methodology behind this project's AI video clipper — how a long
  podcast/interview becomes short, vertical, captioned clips. Use this skill
  whenever working on clip selection, transcript analysis, the ffmpeg render,
  dead-air/pacing removal, speaker reframing, encoding (CPU/GPU), the job
  queue/worker, or when building or explaining any part of the clip-generation
  pipeline — even if the user doesn't say "pipeline" and just mentions clips,
  captions, rendering, scoring moments, ffmpeg, NVENC, or "why is a clip wrong".
  Also use it to answer conceptual/interview questions about how the clipper
  works. It captures the design decisions and the "why", so prefer it over
  re-deriving the flow from scratch.
---

# Clip Pipeline

This project turns a long talking-head video (podcast, interview, commentary)
into several short, vertical (9:16), captioned clips ready for Shorts/Reels/
TikTok. This skill captures **how it works and why it's built this way**, so
changes stay consistent with the design that already produces good output.

The golden rule that explains almost everything: **the whole pipeline is
anchored to word-level timestamps from transcription.** The video pixels are
never analyzed to find clips. Words-in-time is the coordinate system; the LLM
picks *which words*, and ffmpeg cuts at *those times*.

## Architecture (three moving parts)

- **API server** — `backend/app/main.py` (FastAPI). Accepts requests, auth,
  billing, freezes job options, and **enqueues** work. Stays responsive; does no
  heavy rendering.
- **Worker** — `backend/worker.py` + `app/runner.py`. A separate long-running
  process that **claims jobs from the DB queue** and runs the pipeline. Scale by
  running more workers; one render already saturates its cores.
- **Queue + data** — `app/queue.py` (atomic claim/retry/release-stale),
  `app/models.py` (User, Job, QueuedJob, Clip, CreditLedger, Dodo* billing).
  Storage (`app/storage.py`) is local disk or Cloudflare R2.

Why split API from worker: rendering pins the CPU for minutes. If the API did
it, the site would freeze. The queue decouples "accept" from "process".

## The pipeline, in order (and WHY the order matters)

Entry point: `_run_pipeline_locked` in `main.py`. The ordering is deliberate and
driven by **cost** — do the cheap analysis first, pay for the expensive video
download/render only once clips are proven worth making.

1. **Audio-only download first** (`downloader.download_audio`). A few MB, fast.
   The full 1080p video (hundreds of MB) is NOT fetched yet.
2. **Transcribe** (`transcriber.transcribe`, Deepgram) → transcript with
   **word-level timestamps** + utterances. Stored on the Job so re-edits never
   pay for transcription again. `multilingual` for code-switching; `diarize`
   only for the podcast frame.
3. **Prosody** (`energy.analyze`) → per-utterance **loudness (RMS)** and
   **dynamics** from the raw audio. The transcript can't show *how* something was
   said; this finds where the room came alive. It's a bonus signal and degrades
   to a no-op on failure.
4. **Pick clips** (`analyzer.pick_clips`) — see "Clip selection" below.
5. **Charge credits** — per clip, counted here because only now is the real
   number known (ask for 5, get 3 → pay 3). Charged *before* render (where
   compute is spent) but *after* analysis. Remembered so failures **refund**.
6. **NOW download the full video** (`downloader.download_video` → `source.mp4`).
7. **Render** each clip (`editing.py` prepares, `render.render_clip` encodes),
   and build low-res editor **proxies in parallel** (`make_proxy`).

**"Worth making" = four gates passed before the video download:** there is
speech, it's within the length limit, the analyzer found real clips, and the
user can pay. Fail any gate → `workspace.clear_intermediates` and exit, having
never pulled the big file. This is a core cost/latency win — keep new failure
paths *before* step 6 when possible.

## Clip selection — the USP (read `references/clip-selection.md` for depth)

Finding the best clip is a **ranking problem over the whole file**, not one LLM
call. Three layers, each covering the others' blind spots:

- **Acoustic signal** (`energy.py`) grounds taste in delivery, not just text.
- **Rubric-scored LLM judgment** on 6 weighted axes.
- **A selection algorithm** that guarantees whole-file coverage and no dupes.

Key mechanics (constants live in `analyzer.py`):

- **Overlapping windows** — `WINDOW_SIZE = 110` utterances, `WINDOW_OVERLAP = 20`.
  Short windows get read *closely* (long inputs make the model skim the middle);
  overlap means a moment on a boundary is still seen whole by one window.
- **The LLM returns utterance INDEX ranges, not timestamps** (`start_index`/
  `end_index`). Models reliably point at line numbers but hallucinate seconds.
  `resolve_clip_times` maps indices → real seconds via the word timestamps.
- **6-axis weighted score** (`WEIGHTS`): hook 0.26, standalone 0.20, emotion
  0.18, ending 0.16, payoff 0.12, share 0.08. Collapses six 1–10 ratings into
  one comparable number. Hook is weighted highest — if the first 2s don't grab,
  nothing else matters.
- **Nominate widely, select narrowly** — `CANDIDATES_PER_WINDOW = 6` per window
  → pool all → **global re-rank / head-to-head compare** (per-window scores
  aren't calibrated against each other) → **greedy de-overlap** (`_overlaps`) so
  the same moment isn't returned twice → optional sentence-level **refine pass**
  that snaps boundaries to clean starts/ends and trims filler openers.

When touching selection: preserve the index-range contract, the weighting, and
the pool→rank→de-overlap shape. These are the reason the output is good.

## Pacing — dead-air removal (`pacing.py` + `render.py`)

The analyzer gives a rough outer window; within it, silences make a short feel
slow. `pacing.tighten` removes them:

- Detect pauses from **word-gap timestamps**, not audio silence detection — the
  transcript already knows exactly when speech stops/starts. Gap >
  `DEFAULT_MAX_PAUSE` (0.45s; `PODCAST_MAX_PAUSE` 1.50s for monologues, where a
  short gap is emphasis) → dead air, drop it. `HEAD_PAD`/`TAIL_PAD` keep a sliver
  so consonants aren't clipped.
- Returns keep-**segments** + **remapped** words. Critical: cutting time shifts
  everything after the cut earlier, so captions (`remap_words`) and reframe
  keyframes (`remap_time` / `remap_reframe_plan`) MUST be re-timed together or
  they desync.
- **Audio and video are cut together** in the render. `_segments_filter`
  (`render.py`) emits `trim` (video) + `atrim` (audio) per segment at identical
  times, `setpts`/`asetpts` reset each piece's clock, then
  `concat=n=…:v=1:a=1`. Without the setpts reset, the output freezes for exactly
  the gap you removed.

## Render — decode → filter → encode (`render.py`)

Every clip is one ffmpeg run: **DEMUX → DECODE → FILTER → ENCODE → MUX**.

- You must **decode** compressed H.264 to raw pixels because you can't edit
  compressed data; **encode** again because raw is ~180 MB/s. Decode is done by
  ffmpeg's built-in decoder (CPU here — no `-hwaccel` set). **libx264 is an
  encoder only, never a decoder.**
- **Encoder choice** (`_detect_hw_encoder` / `_encoder_flags`): Mac →
  `h264_videotoolbox` (GPU/media engine). Else → `libx264` (CPU). `FFMPEG_THREADS`
  is capped to the container's real CPU quota, not the host's.
- **Filters** per clip: reframe crop to 9:16 (`reframe.plan`, speaker/face-aware
  for podcasts), background blur at quarter-size (cheap), burn the captions (see
  below), overlays/logos (`overlays.py`, `platform_logos.py`), optional censor
  mute spans, and a `watermark_filter` below the caption band for the free tier.

### Captions are a first-class feature (a big part of why clips look good)

Not just "burned subtitles" — this is much of the polish:

- **Animated, word-by-word captions** built as `.ass` (`subtitles.build_ass`),
  with an active-word highlight (karaoke-style timing from word timestamps).
- **Preset registry** (`caption_presets.py`) — one definition per look across
  **six independent axes** (color, highlight, case, position, etc.), rendered to
  both ASS (burn) and CSS (the browser editor) from the same source so the
  preview matches the export. Add a look = one registry entry.
- **Contextual emoji** (`caption_emoji.py`) — the LLM marks only cues that call
  for one emoji (~1 in 3; more reads as noise), validated against what the
  shipped emoji font can actually draw. Any failure → render plain; never
  promise an emoji the export drops, and never fail a paid render over an emoji.
- **Caption placement** (`render.py` `*_caption_margin_v`) clears each platform's
  own UI chrome and adapts to the frame (podcast/fit/split) so captions never
  sit under the Shorts/Reels/TikTok buttons.
- **Translation** (`translation.py`) can replace caption lines for a target
  language while keeping timing.
- **Proxies** (`make_proxy`): `PROXY_WIDTH = 640`, `PROXY_CRF = 30`, `libx264
  veryfast`, dense keyframes for scrubbing. Intentionally low quality — they're
  the *editor* preview, built in parallel so the editor is ready when clips
  appear. The real deliverable is `render_clip`'s high-quality output.

### GPU / NVENC (when asked to make rendering faster on a GPU)

The code has **no NVENC path yet** — only VideoToolbox + libx264. A rented GPU
sits idle unless NVENC is added. To wire it:

- Add detection + flags mirroring the VideoToolbox branch: encode `-c:v
  h264_nvenc -preset p5 -cq 23`, optional decode `-hwaccel cuda`. Keep Mac
  (VideoToolbox) and CPU (libx264) fallbacks intact.
- Best hardware for this workload is an **NVIDIA L4** (media-optimized), on a
  scale-to-zero host (e.g. Modal) so idle GPU cost is $0.
- Honest framing: the LLM analysis (many API calls) and the video download are
  often bigger latency than the encode. For "fast output," also **parallelize
  the analysis window calls and the per-clip encodes**, not just the GPU.

## Editor & re-edits

Clips edit against the 640px proxies (`LiveEditor.jsx`). Save/export
(`editing.rerender_clip`) reuses the **stored transcript** and reads a new time
slice from the kept `source.mp4` — no re-download, no re-transcription. This is
why `source.mp4` is retained as the source of truth.

## Gotchas / invariants to respect

- Don't move paid/expensive work (full video download, render) ahead of the
  "worth making" gates.
- Any time you change clip boundaries or remove time, re-map captions AND
  reframe keyframes, or they desync.
- The LLM contract is index ranges + 1–10 rubric scores; don't switch it to
  raw timestamps.
- `clipper.db` and `storage/` are real user data — cleanup targets seeded/
  intermediate files only, never user clips or the DB.
- Keep CPU/Mac fallbacks when adding any hardware encode path.
