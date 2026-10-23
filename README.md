# Clip Pipeline — a Claude skill

A [Claude](https://claude.com/claude-code) **skill** that captures the full
methodology behind a production AI video clipper: how a long talking-head video
(podcast, interview, commentary) becomes several short, vertical (9:16),
captioned clips ready for Shorts / Reels / TikTok.

It is not a library you call — it is **know-how** that loads into Claude's
context so Claude answers from a real, battle-tested design instead of
re-deriving one. It records not just *what* each step does but **why it's built
that way**, which is what keeps changes from quietly breaking output quality.

## What it covers

- **Architecture** — API / worker / queue split, and why heavy rendering is kept
  off the request path.
- **Audio-first pipeline** — download audio only, transcribe, analyze, and pull
  the full video *last*, once clips are proven worth making (four cost gates).
- **Clip selection (the interesting part)** — a 3-pass, whole-file ranking:
  - overlapping transcript **windows** (close reading beats one long skim),
  - an **acoustic energy** signal (loudness + dynamics) so taste is grounded in
    delivery, not just text,
  - a **6-axis weighted rubric** (hook / standalone / emotion / ending / payoff
    / share) scored by an LLM that returns **utterance index ranges, not
    timestamps**,
  - **pool → head-to-head compare → greedy de-overlap → boundary refinement**.
- **Pacing** — remove dead air from **word-gap timestamps**, cutting audio and
  video together and re-timing captions + reframe keyframes so nothing desyncs.
- **Reframe** — speaker/face-aware crop from 16:9 to 9:16.
- **Captions** — animated word-by-word `.ass` captions, a preset registry,
  contextual emoji, and platform-aware placement.
- **Render** — the `decode → filter → encode` chain, CPU (libx264) vs hardware
  (VideoToolbox) encoding, editor proxies, and how to add **NVENC/GPU**.

## Install

A skill is just a folder with a `SKILL.md`. Copy the `clip-pipeline/` folder
into a skills directory Claude reads:

```bash
# Project-scoped (only inside one repo):
cp -R clip-pipeline /path/to/your-project/.claude/skills/

# Or user-scoped (available everywhere):
cp -R clip-pipeline ~/.claude/skills/
```

## Use

Once installed, it triggers **automatically** when you ask about clips, scoring
moments, captions, ffmpeg, pacing, reframing, encoding, GPU/NVENC, or "how does
the clipper work." You can also load it explicitly in Claude Code:

```
/clip-pipeline
```

### Example prompts that use it

- "Why is this generated clip starting mid-sentence?"
- "Add GPU/NVENC encoding so rendering is faster on a Linux server."
- "Dead air isn't being removed from some clips — where do I look?"
- "Explain how we decide which moment is the best clip." *(great for interviews)*
- "The captions desync after I change a clip's boundaries — why?"

In each case Claude answers from the documented design — the 3-pass selection,
the pacing invariants, the encode chain — instead of guessing.

## A note on the file paths

`SKILL.md` references the file layout of the original implementation
(`backend/app/pipeline/...`) and its tuned constants. Treat those as a concrete
worked example — the *methodology* is the portable part. Adapt the paths and
numbers to your own codebase.

## Structure

```
clip-pipeline/
├── SKILL.md                      # the methodology + the "why" + pointers
└── references/
    └── clip-selection.md         # deep dive on the scoring / ranking algorithm
```

## License

MIT — see [LICENSE](LICENSE). Use it, fork it, adapt it.
