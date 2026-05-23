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
- Language/stack: **TBD at kickoff** (the compositor is Rust; this widget's stack will be decided when work starts).
- Security posture: external/streamed content is untrusted (AiOS F-PROMPT-SAFETY); design accordingly later.

## Next steps (when we start)
- Decide stack + architecture; write PLAN/ROADMAP; scaffold the project; wire it in as an AiOS component.
