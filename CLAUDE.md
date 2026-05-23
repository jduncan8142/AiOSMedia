# CLAUDE.md — AiOSMedia

**Status: planning / not started.** No code exists yet; this repo is a placeholder for a future AiOS widget.

## What this is
AiOSMedia is a planned **canvas widget** for AiOS — an AI-first, Linux-based OS whose desktop is a free-form canvas of widgets rendered by AiOSCanvas (a Smithay/Wayland compositor). Parent project: https://github.com/jduncan8142/AiOS .

AiOSMedia is a combined media-player widget for the AiOS canvas, covering both video and audio. On the video side it plays local video files as well as online content (YouTube and other sites) directly on the canvas. On the audio side it supports streaming services (Spotify and others) alongside local audio files.

## Scope (from the AiOS PARKING_LOT — "Planned Applications & Widgets")
- Video playback of local files rendered directly on the canvas.
- Video playback of online content (YouTube and other sites) on the canvas.
- Audio playback from streaming services (Spotify and others).
- Audio playback of local audio files.

## Conventions inherited from AiOS
- Integrates as a canvas widget hosted by AiOSCanvas; the authoritative cross-project planning docs live in the parent AiOS repo.
- License: Apache-2.0 (AiOS-wide).
- Language/stack: **proposed** — a Rust `aiosmedia-core` library (transport, queue, source resolution, agent seam) with a mature C media engine (libmpv or GStreamer, decided in Phase 0) for decode. Headless core + thin frontends, mirroring AiOSTerminal/AiOSPac. See [PLAN.md](PLAN.md) "Key Decisions".
- Security posture: external/streamed content (media files, manifests, metadata, any embedded web content) is untrusted data under AiOS **F-PROMPT-SAFETY**; decode engine and parsers run sandboxed.
- Credentials: any provider token (Spotify OAuth, YouTube Data API key) goes through **AiOSVault**, never stored locally.

## Planning documents
This repository is **planning-stage**. Read the docs before writing any code:
- [PLAN.md](PLAN.md) — vision, principles, architecture (headless core + canvas frontend, GPU decode/render via AiOSCanvas), scope and non-scope, key decisions + proposed stack, the **online-content legal/technical reality** (YouTube/Spotify APIs vs embeds, accounts/Premium, DRM/Widevine, codec licensing), local-file playback, agent integration, F-PROMPT-SAFETY security, and open questions.
- [ROADMAP.md](ROADMAP.md) — phases mapped to AiOS milestones M1/M2/M3: local playback first, sanctioned online *control* next, in-widget online *playback* a stretch goal.
- `FEATURES.md` (`MD-*` catalog) and `DESIGN.md` — written at Phase 0; not present yet.

The parent **AiOS** repo holds the authoritative cross-project docs (`PLAN.md`, `DECISIONS.md`, `FEATURES.md`, `THREAT_MODEL.md`); read `DECISIONS.md` before reopening any settled AiOS-wide choice.

## Next steps (when we start)
- Confirm the PLAN open questions with Jason (esp. online-playback ambition vs. effort, and engine choice); run the Phase-0 engine/render spike; scaffold the Cargo workspace; write FEATURES/DESIGN; wire it in as an AiOS component.
