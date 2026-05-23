# AiOSMedia — Project Plan

The media-player widget for AiOS: a single canvas object that plays video and
audio — local files and online content — directly on the AiOS canvas.

**Status:** Planning — pre-Phase 0. The repository holds a `CLAUDE.md` stub, the
license, and this document set; there is no widget yet.

## Vision

AiOSMedia is the place media lives on the AiOS canvas. AiOS is an AI-first,
Linux-based operating system that discards the conventional desktop — no windows,
no taskbar, no app launcher — in favour of a free-form **canvas of widgets**
rendered by [AiOSCanvas](https://github.com/jduncan8142/AiOSCanvas), a
Smithay/Wayland compositor. In a world without applications, "open YouTube" or
"play this movie" cannot mean *launch an app*; it has to mean *a media surface
appears on the canvas and starts playing*. AiOSMedia is that surface.

It is **one widget for both video and audio.** The parent project's parking lot
listed a video player and an audio player as two ideas; they collapse into one
component here because they share almost everything that is hard — a playback
engine, a decode-and-render path into the compositor, a transport-control model,
a queue, and the agent-control seam — and differ mainly in their sources and
their on-canvas footprint. One widget, two faces: a framed video surface, and a
compact "now playing" audio surface.

It plays **what the user already has and what is online.** Local video and audio
files play with no account, no network, and no third party — the local-first
path is the one that must always be excellent. Online content — YouTube and
other video sites, Spotify and other streaming services — is layered on top
through each provider's *sanctioned* integration, with a clear-eyed account of
where those integrations require accounts, premium tiers, vendor SDKs, or DRM
that AiOS cannot satisfy. Where a provider offers no lawful programmatic path,
AiOSMedia says so rather than scraping around it.

It is **mostly standalone.** Unlike the inbox or planner widgets, AiOSMedia is
not a view onto the agent's work; it is a tool the user (or the agent on the
user's behalf) drives directly. Its coupling to other widgets is deliberately
low — it depends on AiOSCanvas to host it, on AiOSVault for any provider
credentials, and on the agent for optional voice/queue control, and on nothing
else.

**Near-term:** a canvas widget that plays local video and audio files well, with
GPU-accelerated decode and render through AiOSCanvas. **Long-term:** the full
media surface — sanctioned online video, authenticated music streaming, an
agent-drivable queue, and voice control ("play the next episode", "skip this
track").

## Guiding Principles

1. **Canvas-native, not a window.** AiOSMedia is a widget on the canvas, placed,
   moved, resized, and z-ordered like any other. There is no media-player
   *application* with its own chrome. A feature framed as "the player window"
   is a feature fighting the product.
2. **Local-first, and local is excellent.** The offline path — local files, no
   account, no network — is the baseline experience and the one that must never
   regress. Online sources are additive, never a precondition for the widget to
   be useful.
3. **Don't reinvent the decoder.** Container demuxing, codec decoding, and A/V
   sync are decades-deep, well-solved, and unforgiving to reimplement. Build on
   a mature, hardware-accelerated media engine; spend the novelty budget on
   canvas integration, the agent seam, and the online-source model.
4. **Legality is a design input, not an afterthought.** Each online source is
   integrated only through its sanctioned API/SDK and within its terms of
   service. Where a sanctioned path needs an account, a paid tier, or DRM the
   platform can't provide, that is documented as a constraint the user owns — not
   engineered around. AiOSMedia ships no circumvention.
5. **External and streamed content is untrusted data.** Media files, stream
   manifests, track metadata, captions, thumbnails, and especially any embedded
   web content are *data*, never instructions — AiOS's **F-PROMPT-SAFETY**
   pillar. Parsers and any web-content renderer run sandboxed and treat every
   byte as hostile.
6. **Credentials live in the vault.** Any provider token, OAuth refresh token,
   or API key goes through [AiOSVault](https://github.com/jduncan8142/AiOSVault),
   mediated where the provider's auth model allows. AiOSMedia stores no secret of
   its own.
7. **Headless core, thin frontends.** Playback logic — engine control, source
   resolution, queue, transport state — is a library with no UI, mirroring the
   AiOS-wide library/thin-frontend pattern (AiOSTerminal, AiOSPac). The canvas
   widget is one frontend; a standalone dev window and a headless agent-test
   harness are others.
8. **The agent is a client, not an owner.** The agent can queue, start, pause,
   skip, and query playback through a defined control seam, but it does not embed
   the player. Agent-initiated actions carry provenance and obey the same safety
   model as every other autonomous action.
9. **Touch-first, never touch-blocked.** Transport controls, scrubbing, and
   volume must be fully usable by touch and voice. Keyboard and mouse are
   supported but never required — the AiOS input contract.
10. **Multi-user-ready from day one.** Playback sessions, queues, and any cached
    provider state carry a user scope even though AiOS is single-user through M2.
    Retrofitting a user scope later is the trap the parent project chose to avoid.

## How it fits into AiOS

AiOS is split into a Python core (agent runtime, perception, policy) and a set of
component repositories embedded in the parent
[AiOS](https://github.com/jduncan8142/AiOS) repo. The Rust system components are
AiOSCanvas (the compositor), AiOSFSS (file sync), AiOSVault (secrets), AiOSPac
(packaging), and AiOSTerminal (the terminal). AiOSMedia is a **canvas widget**,
in the same family as the planned calendar, communication, and organizational
widgets — a sibling component that is hosted by AiOSCanvas rather than a headless
daemon supervised by the agent.

Its relationships are few and explicit:

- **AiOSCanvas (host).** AiOSMedia renders into a compositor-provided surface and
  receives input routed by the compositor, exactly as
  [AiOSTerminal](https://github.com/jduncan8142/AiOSTerminal) does. The widget
  contract — how a widget renders, is placed, and receives input — is
  AiOSCanvas's load-bearing internal API (its `C-WIDGET`), and AiOSMedia is a
  consumer of it. The video path additionally wants the compositor's GPU context
  for **zero-copy** decode-to-display (see Architecture).
- **AiOSVault (credentials).** Spotify OAuth tokens, YouTube Data API keys, and
  any other provider secret are stored in and used through AiOSVault, mediated
  where the auth model permits so a raw long-lived token never needs to sit in
  AiOSMedia's memory. AiOSMedia mints no secret of its own.
- **The AiOS agent (optional controller).** The agent can drive playback —
  enqueue, transport control, query "what's playing" — over a control seam, and
  voice commands ("play the next episode", "pause") flow agent → AiOSMedia. This
  is the same agent-as-client posture AiOSCanvas and AiOSTerminal take.
- **Everything else: no coupling.** AiOSMedia imposes no dependency on AiOSFSS,
  AiOSTerminal, or the other widgets. (A user might *sync* their local media
  library with AiOSFSS, but that is the user's filesystem choice, not a
  component dependency.)

AiOSMedia implements the parent feature **F-MEDIA** (minted in the parent
`FEATURES.md` on 2026-05-23) and owns its own sub-feature tags (`MD-*`, e.g.
MD-ENGINE, MD-VIDEO, MD-AUDIO, MD-ONLINE, MD-QUEUE, MD-AGENT, MD-SAFETY). The
parking-lot "video player" and "audio player" items are consolidated into this
one component.

## Architecture

AiOSMedia is a headless core library with swappable frontends — the same pattern
AiOS uses elsewhere (AiOSTerminal's `aiosterm` core, AiOSPac's library core).

```
            AiOS agent runtime (Python, parent repo)
                  ▲ events        │ control ▼
┌──────────────────────────────────────────────────────────┐
│  Frontends (swappable)                                     │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐ │
│  │ Canvas widget  │ │ Standalone     │ │ Headless       │ │
│  │ (AiOSCanvas)   │ │ window         │ │ (tests, agent) │ │
│  │ — the product  │ │ — development  │ │                │ │
│  └────────────────┘ └────────────────┘ └────────────────┘ │
├──────────────────────────────────────────────────────────┤
│  Render / present   video frames → compositor surface;    │
│                      "now playing" UI; transport controls │
├──────────────────────────────────────────────────────────┤
│  aiosmedia core (headless, no UI)                          │
│   playback engine control · A/V sync · transport state ·  │
│   queue / playlist model · source resolution · events     │
├──────────┬─────────────────────────────┬──────────────────┤
│ Source providers                        │ Decode engine    │
│  Local file · YouTube (sanctioned) ·    │ libmpv            │
│  Spotify (SDK) · generic stream         │ + VA-API/VDPAU HW │
├──────────┴─────────────────────────────┴──────────────────┤
│  AiOS services        AiOSVault (tokens) · agent · voice   │
├────────────────────────────────────────────────────────────┤
│  Linux base           DRM/KMS · GPU video decode · ALSA/PW │
└──────────────────────────────────────────────────────────┘
```

- **`aiosmedia` core** — a headless library: it drives the decode engine, owns
  the transport state machine (Idle → Loading → Playing → Paused → Ended/Error),
  performs source resolution (turn a "play this" request into a concrete media
  handle), owns the queue/playlist model, and emits a structured event stream
  (position, state changes, track changes, errors). No rendering, no windowing —
  fully testable headless. Its public API stays independent of the underlying
  engine so the engine can be swapped.
- **Decode engine** — the actual demuxing, codec decoding, and A/V
  synchronization. AiOSMedia does **not** implement these; it embeds a mature
  engine (**libmpv** — decided; see Key Decisions) and asks it to decode using
  the GPU (VA-API on the AMD reference hardware) wherever the codec and hardware
  allow.
- **Source providers** — a trait-based abstraction over where media comes from:
  a local-file provider (the baseline), a sanctioned-YouTube provider, a
  Spotify-SDK provider, and a generic-stream provider. Each provider resolves a
  user/agent request into something the decode engine (or a vendor SDK) can play,
  and declares its own constraints (account required, premium required, DRM,
  region). New providers are added without touching the core.
- **Render / present** — for **video**, the goal is **zero-copy**: the engine
  decodes into a GPU texture (a DMA-BUF / d\-EGLImage handle) that is handed to
  the compositor and composited directly into the widget's region, so frames
  never make a round trip through CPU memory. This needs an explicit contract
  with AiOSCanvas (a new `C-*` capability — see Open Questions and the parent's
  HDR/color-management parking-lot note). For **audio**, "render" is the compact
  now-playing surface (art, title, transport, scrubber). Transport controls and
  scrubbing are touch-first.
- **Frontends** — the **canvas widget** is the product form, hosted by
  AiOSCanvas. The **standalone window** is the development form so AiOSMedia can
  be built and dogfooded before the AiOSCanvas widget contract is ready, and on a
  plain Linux desktop. The **headless** frontend backs tests and lets the agent
  exercise playback without a display. (Whether the standalone window also
  targets Windows for dev convenience, as AiOSTerminal does, is an open question
  — the Linux-only decode stack makes it less clearly worthwhile here.)

### Why a single widget for video *and* audio

The hard parts are shared: one decode engine plays both (libmpv and GStreamer are
both A/V engines), one transport state machine, one queue model, one agent-control
seam, one safety posture for untrusted media. The differences are confined to the
source providers and the *presentation* layer (a framed video rectangle vs. a
compact now-playing card). Splitting into two repos would duplicate the
security-critical decode-and-render surface for no benefit. One core, two
presentations.

## The online-content reality (legal & technical)

This is the part of AiOSMedia where ambition meets constraints, and the
constraints are real. "Play YouTube and Spotify on the canvas" is easy to say and
genuinely hard to do *lawfully and durably*. The honest position: AiOSMedia
integrates each service only through its **sanctioned** path, accepts the
limitations that path imposes, and ships **no circumvention**. The alternatives —
scraping stream URLs, stripping DRM, automating a logged-in web session against
its terms — are rejected as a class. They break without warning, they expose
Jason and any future product to legal risk, and they violate principle 4. Each
major source is treated below as a decision with explicit risks.

### YouTube

- **Sanctioned paths.** (a) The **IFrame Player API** — an official embedded
  player the platform controls; ads, tracking, and playback policy come with it,
  and it expects a browser/WebView environment. (b) The **YouTube Data API v3**
  for search and metadata only — it returns *no* stream URLs and is quota-limited
  with an API key. There is **no** official API that hands you a raw, persistent
  progressive stream URL to feed a native decoder.
- **What this means for a canvas without a browser.** AiOS has no app windows and
  no general browser; the IFrame Player presumes a web runtime. Embedding it
  means hosting a sandboxed WebView/web-content surface *inside* the widget — a
  significant component in its own right, and a meaningful new attack surface
  (treated as untrusted under F-PROMPT-SAFETY). The Data-API-only alternative
  gives clean search/metadata but cannot itself play anything.
- **Rejected.** `youtube-dl`/`yt-dlp`-style stream extraction. It is a perpetual
  cat-and-mouse against YouTube's player, is widely held to violate YouTube's ToS,
  and is the opposite of a durable foundation for a product with corporate
  ambitions. Not shipped.
- **Decision (proposed).** YouTube *search and metadata* via the Data API early;
  YouTube *playback* deferred and gated on solving the sandboxed-web-content
  surface — which is the same capability several other sources need. Treat
  YouTube playback as a stretch goal, not an M-anything commitment.

### Spotify

- **Sanctioned path.** The **Spotify Web API** (search, metadata, playback
  *control*) plus a playback surface. For an app to be the device that actually
  outputs audio, Spotify's model is the **Web Playback SDK** (a browser/EME
  surface) or the (effectively deprecated/closed for new partners) native
  libspotify. The Web Playback SDK **requires Spotify Premium** and runs in a web
  context using **Encrypted Media Extensions (EME) / Widevine** for the protected
  audio stream.
- **What this means.** Playing Spotify audio *through AiOSMedia as the output
  device* requires (a) the user to hold Spotify Premium, (b) an OAuth
  authorization (token in AiOSVault), and (c) a web-content + EME/Widevine
  surface — the same sandboxed-web-content dependency as YouTube, plus a CDM.
  Without those, AiOSMedia can still do **Spotify Connect remote control**: use
  the Web API to control playback on *another* Spotify-certified device the user
  already has (phone, speaker), showing now-playing and transport on the canvas
  without decoding a single protected byte. That is a real, lawful, low-risk
  feature that needs only OAuth + the Web API.
- **Decision (proposed).** Ship **Spotify Connect control** first (Premium-gated,
  OAuth via vault, no DRM in AiOSMedia). Treat **in-widget Spotify audio output**
  as a later, Premium-and-Widevine-gated stretch goal that rides on the same
  web-content/EME surface as YouTube playback.

### DRM, Widevine, and EME

- Protected commercial streaming (Spotify in-app, Netflix-class video, etc.)
  requires a **Content Decryption Module** — in practice **Google Widevine** —
  reached through the browser's **EME** API. Widevine is **proprietary, licensed,
  and not freely redistributable**; obtaining and shipping a CDM in a niche
  Linux OS is a licensing relationship AiOS does not have and may not be able to
  get, and the highest security tier (L1) is hardware-bound. **Software Widevine
  (L3)** typically caps protected video at SD.
- **Decision (proposed).** AiOSMedia ships **no DRM/CDM of its own** in the
  planning horizon. Any DRM-protected source is therefore either (a) controlled
  remotely (Spotify Connect → a certified device that holds the CDM) or (b)
  out of scope. Revisit only if a concrete, licensable CDM path appears.

### Codec licensing (applies to local files too, not just online)

- Some widely-used codecs carry **patent-licensing** obligations: **H.264/AVC**
  and **H.265/HEVC** (MPEG-LA / Access Advance pools), and **AAC**. Linux
  distributions handle this differently — many ship FFmpeg built with these
  decoders and leave licensing to the user's jurisdiction/hardware; others omit
  them. For a hobby/research build the practical exposure is low, but for a
  **shippable product with corporate ambitions** (the parent's stated long-term
  goal) it is a real obligation that must be costed before distribution.
- **Royalty-free codecs** — **VP9**, **AV1**, **Opus**, **FLAC**, **Vorbis** —
  carry no such burden and are preferred where the source offers them.
- **Decision (proposed).** Pre-M3 (research/dogfood), rely on the system FFmpeg
  the chosen engine links, decoding via the GPU where possible, and prefer
  royalty-free codecs. Defer the *distribution* codec-licensing decision (which
  decoders ship in the AiOS LTS image, and how the bill is paid) to the parent's
  Arch→LTS packaging work near M3, where it belongs. Flag it now so it is not a
  surprise then.

### Net effect on scope

The honest consequence: **local playback is fully achievable and is the
foundation; sanctioned online *control* (Spotify Connect, YouTube search) is
achievable and comes next; in-widget online *playback* of major commercial
services is gated on a sandboxed web-content + EME/CDM surface that AiOS does not
yet have and may never be able to license** — so it is planned as a stretch goal
behind that surface, with the user owning the account/Premium/region constraints.

## Local-file playback & codecs

The local path is the baseline and is unencumbered by any of the online drama:

- **Containers:** MP4/MOV, MKV/WebM, AVI, plus audio containers (Ogg, FLAC, WAV,
  MP3, M4A). Demuxing is handled by the engine's container layer (FFmpeg/libav
  under libmpv, or GStreamer's demuxers).
- **Video codecs:** H.264, H.265, VP9, AV1 — **hardware-decoded via VA-API** on
  the AMD reference GPU wherever the codec/profile is supported, with software
  fallback. AV1 and VP9 are preferred (royalty-free); H.264/H.265 carry the
  licensing note above.
- **Audio codecs:** AAC, MP3, Opus, FLAC, Vorbis, PCM. Opus/FLAC/Vorbis are
  royalty-free and preferred.
- **Audio output:** through the system audio server (**PipeWire** on a current
  Arch base, with ALSA underneath).
- **Subtitles/metadata:** subtitle tracks (SRT/ASS/embedded) and tag metadata are
  parsed as **untrusted data** (a malformed subtitle file is a classic parser
  exploit and a text-injection vector) and sandboxed accordingly.

The local-file source provider, the engine integration, and the decode-to-canvas
path are the entirety of the first usable milestone.

## Agent integration

AiOSMedia is mostly user-driven, but the agent is a first-class *client*:

- **Control seam.** A defined control interface — the same architectural shape as
  AiOSCanvas's and AiOSVault's Unix-socket control planes — lets the agent
  enqueue media, issue transport commands (play/pause/seek/skip/next/previous),
  set volume, and query state ("what's playing", queue contents). The agent runs
  in the parent Python project and reaches AiOSMedia over this seam; it never
  embeds the player.
- **Voice.** Voice commands ("play the next episode", "skip this track", "pause
  the music", "play my focus playlist") are interpreted by the AiOS voice/agent
  stack and arrive as structured control calls. AiOSMedia owns *playback*, not
  speech recognition.
- **Events out.** AiOSMedia emits state/position/track-change/error events the
  agent can observe — e.g. to know a track ended, or to surface a playback error
  to the user.
- **Provenance and safety.** An agent-initiated action carries provenance, and
  any action prompted by *external* content (e.g. a track suggestion that
  originated in an email) is treated as cross-domain and follows
  F-PROMPT-SAFETY's confirmation model. The agent driving the player must not
  become a path by which untrusted content exfiltrates or surprises the user.

## Security considerations (F-PROMPT-SAFETY)

The parent threat model's defining surface is **hostile external content**, and a
media widget is squarely on it. AiOSMedia ingests attacker-controllable bytes
continuously: media files, stream manifests, container/codec data, subtitles,
ID3/Vorbis/MP4 tags, thumbnails, and — if in-widget online playback is ever
built — arbitrary web content. The posture:

- **All media and metadata are data, never instructions.** Titles, tags,
  captions, and manifest fields never reach the privileged agent planner as
  instructions; they are structured data on the same footing as email or web
  content (parent THREAT_MODEL §7.5).
- **Sandbox the parsers and the decode engine.** Container demuxers and codec
  decoders are a perennial source of memory-corruption CVEs (FFmpeg's history is
  instructive). The decode engine and parsers run sandboxed, treating every file
  as hostile input — consistent with the parent's "sandbox parsers/renderers from
  first use" requirement.
- **The web-content surface, if built, is the highest-risk element.** Any
  embedded WebView (for YouTube IFrame / Spotify Web Playback) is a large, hostile
  attack surface and an injection vector. It is sandboxed, script from external
  content is never trusted, and it is treated as a distinct trust boundary. Its
  cost and risk are a major reason in-widget online playback is a stretch goal,
  not a baseline.
- **Credentials never leak through the player.** Provider tokens live in
  AiOSVault, mediated where possible; AiOSMedia holds the minimum, briefly, and
  zeroizes. A compromised or injected player must have nothing long-lived to
  steal.
- **Network egress is constrained.** Online sources talk only to their provider's
  endpoints; novel destinations are subject to the parent's egress-allowlist /
  confirmation model so a malicious manifest can't redirect playback to an
  exfiltration endpoint.
- **Stack choice serves safety.** Mandating Rust for the core (see Key Decisions)
  keeps the control/queue/source-resolution logic memory-safe; the unavoidable C
  in the decode engine (FFmpeg/libav) is confined behind the engine boundary and
  sandboxed, rather than spread through the codebase.

## Key Decisions

### Inherited from the AiOS project (confirmed — see parent `DECISIONS.md`)

- **Apache 2.0** license across all code.
- **Canvas-native**: a widget hosted by AiOSCanvas — no windows, no chrome.
- **Touch is primary**; keyboard and mouse supported but never required.
- **Multi-user-ready** from the start; AiOS runs single-user through M2.
- **F-PROMPT-SAFETY**: all external/streamed content is untrusted data.
- **Credentials go through AiOSVault** (F-VAULT), mediated where possible.

### AiOSMedia-specific — proposed, to confirm in Phase 0

- **Stack: Rust core, mature media engine for decode.** *Proposed.* The
  `aiosmedia` core (transport state, queue, source resolution, control seam,
  event stream) is **Rust** — it sits next to the compositor's privilege boundary
  and handles untrusted input, so the AiOS rule "Rust for system/security-critical
  components" applies, and it keeps the core memory-safe and consistent with its
  sibling components. Decode itself is delegated to a **mature C media engine**,
  not reimplemented (principle 3). *Rationale:* reimplementing demux/codec/sync is
  out of the question; the value AiOSMedia adds is canvas integration, the agent
  seam, and the source model — all of which are Rust.

- **Decode engine: libmpv.** *Decided (2026-05-23, per Jason).* **libmpv** is a
  single, batteries-included, embeddable A/V engine with first-class hardware
  decoding (VA-API), a clean render API (`render` / OpenGL / `libplacebo`) suited
  to compositing into a GPU surface, and good Rust bindings — the
  lower-integration-effort path to excellent local playback. GStreamer was the
  considered alternative (more modular, richer `gstreamer-rs` bindings, a pipeline
  model that might map onto per-source providers + DMA-BUF export) but carries
  more assembly for no decisive benefit here; both ultimately use FFmpeg/libav for
  the codecs. A Phase-0 spike still *validates* decode-to-DMA-BUF, VA-API, and
  A/V-sync quality on the reference hardware, but the engine itself is settled.

- **Video render: zero-copy DMA-BUF into the compositor.** *Proposed, depends on
  AiOSCanvas.* The engine decodes into a GPU texture exported as a DMA-BUF that
  AiOSCanvas composites directly into the widget region — no CPU round-trip. This
  requires a new AiOSCanvas widget capability (importing an externally-produced
  GPU surface). If that capability is not ready, the fallback for early
  development is a copy-based path in the standalone frontend. *Rationale:*
  software-copying decoded video frames per-frame is the difference between smooth
  and unwatchable; on the AMD reference GPU the right answer is zero-copy.

- **Online sources: deferred to a later discussion.** *Deferred (2026-05-23, per
  Jason).* The online-source model as a whole — provider playback and control —
  is parked until a dedicated discussion; it is **not** near-term scope. The
  standing guardrail for when it *is* taken up: sanctioned APIs only — YouTube via
  Data API (metadata) + IFrame (needs a web surface); Spotify via Web API +
  Connect control first, in-widget output behind Premium+EME; no stream-ripping,
  no DRM-stripping, no ToS-violating automation. *Rationale:* principle 4 and a
  clean legal posture remain non-negotiable, but the scope and sequencing are a
  later call, not settled here. (Full background above.)

- **No DRM/CDM shipped.** *Proposed (firm for the planning horizon).* AiOSMedia
  embeds no Widevine/CDM. Protected content is remote-controlled (Spotify
  Connect) or out of scope. *Rationale:* Widevine is proprietary, licensed, not
  freely redistributable, and AiOS has no such relationship.

- **Codecs / format policy: deferred to a later discussion.** *Deferred
  (2026-05-23, per Jason).* Which codecs ship, the hardware-decode paths, and the
  H.264/H.265/AAC patent-licensing posture are all parked for a dedicated
  discussion rather than settled here. Current leaning (not a commitment): the
  engine's system FFmpeg with GPU decode, preferring royalty-free AV1/VP9/Opus/
  FLAC, with the H.264/H.265/AAC licensing call surfaced no later than the M3
  packaging work. *Rationale:* it's a real distribution-time cost that should be
  owned, but the decision does not need to be made now.

- **Layout: cargo workspace (core lib + frontends).** *Proposed.* An
  `aiosmedia-core` library plus an `aiosmedia` binary/frontends, mirroring
  AiOSVault/AiOSCanvas/AiOSTerminal so the security-relevant core is independently
  auditable. Final call in Phase 0. Rust edition 2024; toolchain pinned to match
  the sibling repos (currently 1.94).

## Repository layout (planned)

```
AiOSMedia/
├─ Cargo.toml              # cargo workspace (recommended)
├─ Cargo.lock              # committed
├─ rust-toolchain.toml     # channel pinned to the sibling repos (1.94)
├─ Makefile                # build / release / test / lint / fmt / clean
├─ LICENSE                 # Apache-2.0 (present)
├─ NOTICE
├─ README.md               # present (stub)
├─ CLAUDE.md               # present — agent guidance, links this doc set
├─ PLAN.md ROADMAP.md FEATURES.md TODO.md   # this document set (PLAN+ROADMAP now)
├─ DESIGN.md               # engine integration, render path, source providers
├─ .gitignore              # present
├─ aiosmedia.example.toml  # widget tunables (engine opts, cache, providers)
├─ .github/workflows/      # CI: fmt, clippy, test
├─ crates/
│  ├─ aiosmedia-core/      # engine control, transport, queue, sources (lib)
│  └─ aiosmedia/           # binary — standalone frontend + canvas-widget entry
└─ tests/                  # integration tests, sample media fixtures
```

`FEATURES.md` (the `MD-*` catalog) and `DESIGN.md` are written at Phase 0; this PR
delivers `PLAN.md` and `ROADMAP.md`.

## Proposed dependencies

- **House style, shared with the sibling repos:** `anyhow`, `thiserror`, `clap`
  (derive), `serde`/`serde_json`, `toml`, `tracing`/`tracing-subscriber`.
- **Decode engine binding:** `libmpv`-family bindings *or* `gstreamer-rs` —
  decided with the engine choice in Phase 0.
- **Async / control seam:** `tokio` for the control plane and network source
  providers (the engine itself runs on its own threads/event model).
- **Online providers:** an HTTP client (`reqwest`) for the YouTube Data and
  Spotify Web APIs; an OAuth helper for the Spotify authorization flow (tokens
  persisted via the AiOSVault client, **not** locally).
- **GPU surface interchange:** DMA-BUF / EGL handling for the zero-copy render
  path — exact crates determined by the engine and the AiOSCanvas widget
  contract.
- **Memory hygiene:** `zeroize`/`secrecy` for any transient credential material.
- **Dev:** `tempfile`, sample media fixtures; the standalone frontend's
  windowing crate (e.g. `winit`) for the dev loop.

The C decode engine (FFmpeg/libav, via libmpv or GStreamer) is a **system
dependency**, provided by the Arch base / AiOS image, not vendored.

## Phases & milestones

Phase and milestone planning lives in [ROADMAP.md](ROADMAP.md). In short: Phase 0
scaffolds the project and runs the engine/render spike; the first usable
deliverable is **local video + audio playback** on the canvas; sanctioned online
*control* (Spotify Connect, YouTube search/metadata) follows; in-widget online
*playback* is a stretch goal gated on a sandboxed web-content + EME surface AiOS
does not yet have. Phases track AiOS milestones **M1 (Q3 2026), M2 (Q4 2027),
M3 (Q4 2028)** — AiOSMedia does not set its own milestones.

## Risks

- **Solo, part-time timeline.** AiOSMedia is one of several AiOS components built
  on evenings and weekends. Mitigation: the local-playback core is the floor and
  the easiest to narrow; online sources are independently deferrable.
- **The video render path is cross-repo and hard.** Zero-copy decode-to-canvas
  needs an AiOSCanvas widget capability that does not exist yet (importing an
  external GPU surface). A wrong or late contract ripples into both repos.
  Mitigation: prove it with a Phase-0 spike; keep a copy-based fallback for the
  standalone frontend so AiOSMedia is not blocked on AiOSCanvas.
- **Online integrations are fragile and legally bounded by others.** Provider
  APIs, terms, quotas, and SDKs change at the providers' discretion; Premium/DRM
  requirements are outside AiOS's control. Mitigation: sanctioned paths only,
  graceful degradation (Connect control when in-widget output isn't possible),
  and explicit user-owned constraints — never circumvention.
- **DRM is likely unsolvable for in-widget playback.** A licensable Widevine path
  for a niche Linux OS may simply not exist. Mitigation: design so the product is
  fully valuable *without* it (local + Connect control), and treat in-widget
  protected playback as a true stretch goal.
- **Codec licensing at distribution.** H.264/H.265/AAC carry patent obligations
  that bite when AiOS ships (M3). Mitigation: prefer royalty-free codecs; surface
  the decision now; resolve it in the parent's LTS packaging work, not silently.
- **VM graphics can't validate the video path.** Like AiOSCanvas, the dev VMs are
  software-rendered (VMSVGA, 16 MB VRAM) and cannot exercise VA-API hardware
  decode or smooth video. Physical-hardware validation on the AMD reference GPU
  is required before any milestone sign-off.
- **Decode engines drag in a large C surface.** libmpv/GStreamer/FFmpeg are big C
  codebases with a CVE history. Mitigation: confine them behind the engine
  boundary, sandbox decode, keep the Rust core in control, treat all media as
  hostile.

## Open Questions

Decisions that need Jason's (or the parent project's) input:

1. ~~F-MEDIA in the parent, and merging the two parking-lot items.~~ **RESOLVED
   (2026-05-23, per Jason):** **F-MEDIA** is minted in the parent `FEATURES.md`,
   and the parking-lot "video player" + "audio player" items are consolidated
   into this one component (AiOSMedia).
2. **Online-playback ambition vs. effort.** **DEFERRED (2026-05-23, per Jason)** —
   parked for a later discussion; not being resolved now. Is in-widget
   YouTube/Spotify playback worth building the sandboxed web-content + EME surface
   it requires — a large component and a major attack surface — or is **sanctioned
   control + local files** the right long-term product, with in-widget online
   playback permanently a stretch goal? This is the single biggest scope decision.
3. ~~Decode engine: libmpv vs. GStreamer.~~ **RESOLVED (2026-05-23): libmpv** —
   per Jason. A Phase-0 spike still validates VA-API / DMA-BUF / A-V sync on the
   reference hardware, but the engine is settled (see Key Decisions).
4. **Spotify scope.** **DEFERRED (2026-05-23, per Jason)** — parked with the
   online-sources discussion. Is **Spotify Connect remote control** (Premium-gated,
   no DRM, OAuth via vault) the right first/only Spotify integration, deferring
   in-widget audio output indefinitely? Connect control is cheap and lawful;
   in-widget output needs Premium + Widevine.
5. **Codec / format policy for the shipped product.** **DEFERRED (2026-05-23, per
   Jason)** — parked for a later discussion. Which codecs ship in the AiOS LTS
   image, the hardware-decode paths, and who bears the H.264/H.265/AAC licensing?
   Deferrable to the M3 packaging work, but it is a real cost that should be
   acknowledged as owned.
6. **Standalone-frontend platform.** Should the dev/standalone frontend target
   **Windows** as AiOSTerminal does, or stay Linux-only given the Linux-bound
   decode/VA-API stack makes a Windows port less clearly worthwhile?
7. **AiOSCanvas video-surface contract.** Confirm AiOSCanvas will expose a widget
   capability to import an externally-decoded GPU surface (DMA-BUF) — this is a
   new `C-*` capability and a dependency for the zero-copy video path; needs
   coordination with the AiOSCanvas roadmap.

## Change Log

| Date | Change | Rationale |
|------|--------|-----------|
| 2026-05-22 | Initial AiOSMedia planning document set created (PLAN, ROADMAP) | Repository kickoff; derived from the parent AiOS planning documents and the PARKING_LOT "Planned Applications & Widgets" entries (video + audio players, combined per kickoff brief) |
| 2026-05-23 | Locked in **libmpv** as the decode engine and minted **F-MEDIA** in the parent (Open Questions #1, #3 resolved) | Plan approved by Jason; engine choice and parent feature tag settled |
| 2026-05-23 | **Deferred codecs / format policy and online sources** to a later discussion (Key Decisions + Open Questions #2, #4, #5 parked) | Per Jason: keep the engine + architecture decided, but park codec/format and online-source scope until a dedicated discussion |
