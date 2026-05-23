# AiOSMedia — Roadmap

The time-ordered delivery plan for the AiOS media-player widget.

For vision, architecture, the online-content reality, and the technology choices,
see [PLAN.md](PLAN.md). The feature catalog referenced by tag (`MD-*`) is written
in `FEATURES.md` at Phase 0; concrete tasks live in `TODO.md`.

## How this roadmap relates to AiOS

AiOSMedia is one component of the AiOS project, and its schedule is **subordinate
to the parent AiOS milestones** — M1 (booting prototype, Q3 2026), M2 (daily
driver, Q4 2027), M3 (alpha, Q4 2028). AiOSMedia does not set its own milestones.

AiOSMedia is **not on the critical path for AiOS M1**: M1 is a booting compositor
with one interactive widget and no AI, and a media player is not that widget. So
AiOSMedia is built in the time around the compositor and agent work, and its
canvas-hosted phases are gated on AiOSCanvas exposing the widget contract (and,
for video, a GPU-surface-import capability). Dates below are deliberately
approximate and inherit the parent's solo-developer posture: **when a date is at
risk, narrow the phase's scope rather than slip the date.**

The standing scope discipline: **local playback is the floor and must be
excellent; sanctioned online *control* comes next; in-widget online *playback* is
a stretch goal** behind a sandboxed web-content + EME surface AiOS does not yet
have (see PLAN "The online-content reality"). Resist letting the online ambition
delay a solid local player.

## Milestone overview

| Milestone | Date | Phases | What AiOSMedia delivers |
|-----------|------|--------|--------------------------|
| **M1 — Booting prototype** | Q3 2026 | P0 (partial) | Not on the M1 critical path. At most: project scaffold, the engine/render spike, and a standalone local player running on the dev hardware. No canvas integration required for M1. |
| **M2 — Daily driver** | Q4 2027 | P1–P4 | Local video + audio playing in an AiOSCanvas widget with GPU decode; transport + queue; agent control seam and voice; sanctioned online *control* (Spotify Connect, YouTube search/metadata). |
| **M3 — Alpha release** | Q4 2028 | P5–P6 | Hardening, performance, accessibility; the codec-licensing posture resolved for the shipped image; *if pursued*, in-widget online playback behind the sandboxed web-content + EME surface. |

## P0 — Foundations & Spike · *heavy*

**Goal:** turn the bare repository into a buildable Rust project, and de-risk the
two hardest unknowns — the decode engine and the decode-to-canvas render path —
before committing to them.

- Scaffold the Cargo workspace (`aiosmedia-core` + `aiosmedia`),
  `rust-toolchain.toml` (pinned to the sibling repos, currently 1.94), `Makefile`,
  `NOTICE`, `.gitignore`, `aiosmedia.example.toml`, and CI (GitHub Actions: `fmt`,
  `clippy --all-targets -- -D warnings`, `test`).
- **Engine spike** — stand up **libmpv** *and* a minimal **GStreamer** pipeline on
  the AMD reference hardware; play a local file via each; measure VA-API hardware
  decode, A/V sync quality, and integration effort. **Decide the engine.**
- **Render spike** — get decoded video frames onto a GPU surface zero-copy
  (decode → DMA-BUF → present), first in a standalone window, to prove the path
  the canvas widget will later need.
- Resolve the Phase-0 decisions in [PLAN.md](PLAN.md): engine choice, crate
  layout, standalone-frontend platform, and the AiOSVault client integration
  shape for provider tokens.
- Write `FEATURES.md` (the `MD-*` catalog) and `DESIGN.md` (engine integration,
  render path, source-provider trait).

**Exit:** `make build` / `make test` / `make lint` succeed; the engine is chosen
on evidence; a standalone window plays a local file with hardware decode; the
design is written down.

## P1 — Local Playback Core · *heavy*

**Goal:** the headless `aiosmedia-core` plays local media well, with a standalone
frontend to dogfood it before the canvas is ready.

- `MD-ENGINE` — wrap the chosen engine: load, play, pause, seek, stop; the
  transport state machine (Idle → Loading → Playing → Paused → Ended/Error);
  the structured event stream (position, state, errors).
- `MD-LOCAL` — the local-file source provider: containers (MP4/MOV, MKV/WebM,
  Ogg, FLAC, WAV, MP3, M4A), video codecs (H.264/H.265/VP9/AV1, GPU-decoded),
  audio codecs (AAC/MP3/Opus/FLAC/Vorbis/PCM), subtitle and tag parsing **as
  untrusted data**, audio out via PipeWire.
- `MD-VIDEO` / `MD-AUDIO` — the two presentations in the standalone frontend: a
  framed video surface, and a compact now-playing surface; touch-first transport
  controls and scrubbing.
- `MD-SAFETY` (begin) — sandbox the decode engine and parsers from first use;
  treat all media/metadata as hostile input.

**Exit:** the standalone `aiosmedia` plays local video and audio with hardware
decode on the reference hardware, controllable by touch/keyboard. No canvas, no
online sources, no agent yet.

## P2 — Queue & Agent Seam · *moderate*

**Goal:** a queue model and the control interface the agent will drive — built
headless so it is testable without a display or the compositor.

- `MD-QUEUE` — the playlist/queue model: enqueue, reorder, next/previous, repeat,
  shuffle; user-scoped (multi-user-ready) from the start.
- `MD-AGENT` — the control seam (Unix-socket control plane, same shape as
  AiOSCanvas/AiOSVault): enqueue, transport commands, volume, and state queries
  in; events out. Provenance on every control call; cross-domain confirmation for
  actions prompted by external content (F-PROMPT-SAFETY).
- Headless frontend for tests and agent exercise.

**Depends on:** the agent control-seam shape (align with AiOSCanvas/AiOSVault).
This is the first phase coupled to a parent deliverable — sequence accordingly.

**Exit:** the agent (or a test harness) can drive playback and read state over the
seam; the queue works headless.

## P3 — Canvas Integration · *moderate* · tracks AiOSCanvas

**Goal:** AiOSMedia runs *inside* AiOS as a real canvas widget.

- `MD-CANVAS` — integrate as an AiOSCanvas widget: render into a
  compositor-provided surface, input routed by the compositor, user-scoped
  playback sessions.
- **Video surface contract** — consume the AiOSCanvas capability to import an
  externally-decoded GPU surface (DMA-BUF) for zero-copy video; fall back to the
  copy path if unavailable. (PLAN Open Question 7 — a cross-repo dependency.)
- Placement, resize, and z-order behaviour as a canvas object; the now-playing
  audio surface and the framed video surface as canvas presentations.

**Depends on:** AiOSCanvas exposing its widget model (its P2/`C-WIDGET`) and a
GPU-surface-import capability. Timing tracks AiOSCanvas, not just the calendar.

**Exit:** local video and audio play inside an AiOSCanvas widget on the reference
hardware, placed and controlled on the canvas by touch.

## P4 — Sanctioned Online Sources & Voice · *moderate* · **→ M2**

**Goal:** the AiOS milestone M2 — the lawful, low-risk online integrations that
need no DRM, plus voice control.

- `MD-ONLINE` (control tier) — **Spotify Connect remote control** (Premium-gated,
  OAuth via AiOSVault, no DRM in AiOSMedia): now-playing + transport for playback
  on the user's certified devices. **YouTube search/metadata** via the Data API.
- `MD-AGENT` / voice — voice commands ("play the next episode", "skip", "pause
  the music", "play my focus playlist") arriving as structured control calls from
  the AiOS voice/agent stack.
- Provider-credential flow fully through AiOSVault, mediated where the auth model
  permits; egress constrained to provider endpoints.

**Exit (= AiOSMedia's M2 contribution):** with the agent runtime and AiOSVault,
the canvas widget plays local media, controls Spotify playback on the user's
devices, surfaces YouTube search, and obeys voice transport commands.

## P5 — Hardening & Performance · *moderate* · **→ M3**

**Goal:** alpha-quality — the parent milestone M3.

- `MD-PERF` — frame-pacing and A/V-sync targets; smooth playback under canvas
  load; memory and decode-power behaviour on the reference GPU.
- `MD-SAFETY` (complete) — full sandboxing review of the decode/parser surface;
  egress-allowlist enforcement; external review of the untrusted-content handling.
- Accessibility passes (captions, transport reachability by touch/voice);
  validation on physical touch hardware (the dev VMs cannot exercise VA-API or
  smooth video — see PLAN Risks).
- **Codec-licensing posture resolved** for the shipped AiOS image, coordinated
  with the parent's Arch→LTS packaging work (PLAN Open Question 5).

## P6 — In-Widget Online Playback · *heavy* · **stretch, post-M3 / if pursued**

**Goal:** *only if* PLAN Open Question 2 is answered "yes" — play protected online
content directly in the widget. This phase is explicitly contingent and may never
be scheduled.

- Sandboxed **web-content surface** (WebView) for the YouTube IFrame Player and
  the Spotify Web Playback SDK — a large component and a major new attack surface,
  treated as a distinct trust boundary under F-PROMPT-SAFETY.
- **EME / CDM dependency** — gated entirely on a *licensable* Widevine path
  existing for AiOS; if none does, this phase does not happen. No circumvention.
- In-widget Spotify audio output (Premium + Widevine) and in-widget YouTube
  playback (IFrame).

**Reality check:** PLAN treats this as a stretch goal precisely because the
web-content surface is costly and the DRM/CDM path may be unobtainable for a niche
Linux OS. The product is designed to be fully valuable *without* this phase
(local files + sanctioned control).

## Cross-component dependencies

- **AiOSMedia is not on the M1 critical path** — M1 is the compositor with one
  widget and no AI; AiOSMedia is built around it.
- **P3 (canvas) depends on AiOSCanvas** exposing its widget model *and* a
  GPU-surface-import capability for zero-copy video — a new `C-*` capability to
  coordinate with the AiOSCanvas roadmap.
- **P2/P4 (agent + voice) depend on the parent agent runtime** and the
  control-seam shape shared with AiOSCanvas/AiOSVault.
- **Online sources depend on AiOSVault** for provider credentials (mediated where
  possible) — so the online tiers track AiOSVault's daemon/control-plane
  availability (its V2).
- **No dependency on AiOSFSS or AiOSTerminal.** (A user may sync their media
  library with AiOSFSS, but that is a filesystem choice, not a component coupling.)

## Change Log

| Date | Change | Rationale |
|------|--------|-----------|
| 2026-05-22 | Initial roadmap created | Repository kickoff; phases aligned to the parent AiOS milestones M1/M2/M3, with local playback first and online integrations layered behind it |
