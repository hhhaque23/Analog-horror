# HANDOFF — Claude context for DEAD AIR

Read this first if you are an AI assistant (or a human) picking up this repo.

## What this is

**DEAD AIR** is a single-file analog horror game. You play the overnight
master-control operator at WKRX-TV: watch six channels, purge intrusions, and
survive from 12:00 AM to 6:00 AM. The entire game — rendering, VHS effects,
monsters, music, sound design — is generated procedurally at runtime.

**Hard constraints (do not break these):**

- **One file.** All gameplay code lives in `index.html`. No build step, no
  bundler, no npm.
- **Zero assets.** No image/audio/font files. Visuals are canvas-drawn;
  audio is synthesized with WebAudio. If you add content, generate it in code.
- Canvas is fixed at **640×480** (4:3, CRT look), CSS-scaled to fit.
- Audio can only start after a user gesture — the BOOT screen exists to
  capture that first click/keypress before anything needs sound.

## File map

| File | Purpose |
|---|---|
| `index.html` | The whole game: CSS (CRT bezel/overlay), then one `<script>` |
| `README.md` | Player-facing docs + Steam shipping notes |
| `HANDOFF.md` | This file |

## Architecture of `index.html`

The script is organized top-to-bottom in sections (search for the banner
comments):

1. **utility** — `rnd/ri/clamp/lerp/pick`.
2. **audio** — `initAudio()` builds persistent beds: 60 Hz mains hum,
   looping white-noise static (`setStaticLevel`), dread drone
   (`setDroneLevel`), whisper bed (`setWhisperLevel` — band-passed noise with
   an LFO-wandering formant). One-shots: `beep`, `stinger` (scare hit),
   `scream`, `screech` (tracking slip), `thump` (heartbeat),
   `easTone`, `phoneRingTone`, `evilRingTone` (slower/lower — the tell),
   `silenceScare` (ducks master gain, then slams back).
3. **MOSH studio intro audio** — `studioAudio()` schedules the whole
   fanfare on the WebAudio timeline at once (hiss ramp, sub-booms at each
   `LAND[i]` letter landing, saw-chord swell, sine ident pad, and a
   detune ramp starting at t+7.2 for the corruption pitch-fall).
   `stopStudioAudio()` kills it on skip.
4. **game state** — `ST` enum and flow:
   `BOOT → STUDIO → MENU → INTRO → PLAY ⇄ EAS → SCARE → DEAD` or `WIN`.
   The phone is **not** a state — it is an overlay inside PLAY so gameplay
   never pauses.
5. **input** — `advance()` is the shared "any key / click" state dispatcher;
   PLAY-specific keys (1–6, SPACE, E) are in the keydown handler.
6. **anomaly system** — the core loop. See below.
7. **scripted events** — `buildEvents()` returns the night's timeline;
   `scriptTick()` fires entries once when `gameMin` passes `t`, plus hourly
   chime lines (`HOURLINES`). Phone calls come from the `CALLS` table.
8. **drawing helpers** — `drawStatic`, `text`, `fmtClock`, `corruptStr`
   (OSD glyph rot above threat .7), `trackingLines` (VHS tear bands).
9. **monsters** — see below.
10. **MOSH studio intro visuals** — `drawStudio(t)` + `drawMoshLogo`.
11. **channel renderers** — `chTestCard`, `chWeather`, `chCamera` (lot=3 /
    hall=4), `chButtons`; dispatched from `drawPlayChannel`.
12. **screens** — `drawBoot/Menu/Intro/EAS/Dead/Win`.
13. **anomaly overlays** — `drawAnomalyOverlay` per kind.
14. **main loop** — `frame()`: fixed state switch, OSD/trackbar DOM text,
    ambient flicker.

## The game loop (anomaly system)

`anomalies` maps channel → `{age, kind, resp, fuse, trackTime, warned}`.

| resp | kinds | How it's cleared | Fuse |
|---|---|---|---|
| `track` | `figure` `face` `text` `invert` | tune in + hold SPACE for `trackTime` (2.4s) | 32s |
| `track` | `surge` | same, but `trackTime` 1.1s | **10s** |
| `stare` | `stare` | **look away** — grows at 2.6×/s while watched, starves at .85×/s when ignored; removes itself at age ≤ 0 | 26s |

- Spawning: `nextAnomalyIn` counts down; interval is
  `clamp(16 - hour*2.2, 5, 16)` (min 5s late-night). `nextTarget` is chosen
  **at schedule time** so Channel 6 can leak it (see below).
- Kind odds after the early game: 22% stare (hour > .8), 18% surge
  (hour > 1.6), rest classic.
- SPACE tracking randomly **slips** (`slipT`, ~.45/s chance, not on surges):
  progress bleeds and `screech()` plays — keeps tracking active, not passive.
- A fuse expiring = containment failure (`failAnomaly`): strike, −12
  integrity, stinger. **3 strikes → `triggerScare()` → death.**

### Threat / dread

`threat = clamp(hour/6*.42 + strikes*.18 + activeAnomalies*.06 + ch6Heat + dread, 0, 1)`

Threat drives: drone volume, hum gain, grain/tracking wear, OSD glyph
corruption, subliminal 1-frame face flashes (threat > .5), heartbeat
(threat > .55, accelerating), and monster aggressiveness. `dread` is added by
bad choices (answering the evil call, the 5:30 beat) and decays slowly.

### Channel 6 (risk/reward)

Watching static channel 6 accrues `ch6WatchT`: whispers fade in, `ch6Heat`
raises threat, and after 3s the static **names the next target channel**
(`nextTarget`) — intel for danger. The dead operator stands in the static
after ~1:30 AM and rotates to face the viewer over the night
(`opFacing = clamp((gameMin-120)/240, 0, 1)` — the `facing` param of
`drawFigure` controls eye visibility).

### Scripted timeline (gameMin, 1 real second = 1 minute)

| t | Event |
|---|---|
| 45 | mains hum shifts to 63 Hz |
| 60 | EAS broadcast (state `EAS`, ~13s) |
| 95 | phone call 0 (Marla — friendly) |
| 150 | phone call 1 (warning) |
| 185 | Mr. Buttons corrupts (`buttonsCorrupt`) |
| 215 | **phone call 2 — the entity** (`evil:true`; slow low ring + glitched prompt are the tells; answering = 2 anomalies + dread + −8 integrity; ignoring = +4 integrity) |
| 250 | blackout: 3 simultaneous intrusions |
| 290 | `silenceScare()` — audio drops out, then slams |
| 330 | final dread bump |
| 360 | WIN |

### Scoring

`integrity` starts at 100: −12/strike, −.12/s per active anomaly, +2 per
restore, ±phone outcomes. Grade at end (`finishGrade`): S ≥92, A ≥80, B ≥65,
C ≥45, else D; death = F. Last grade persists to
`localStorage["deadair_last"]` and shows on the menu.

## The monsters

- **`drawFigure(x, y, scale, dark, facing)` — the Tall One.** Emaciated,
  head tilted wrong (tilt grows with threat), pinprick eyes that track
  toward screen center and blink, splayed fingers, slow breathing scale.
  Movement style: unnaturally still with rare sideways **teleport twitches**
  (quantized-time hash, not sine wobble). `facing` 0..1 fades the eyes in —
  used for the channel-6 operator turning around.
- **`drawCrawler(x, y, scale)`** — all-fours thing with knees above its
  body; renders in stop-motion (time quantized to 130ms steps, per-step
  seeded limb poses). Skitters across camera feeds after 2 AM.
- **`drawWatcher(x, y, n, sev)`** — n pairs of blinking eyes opening in dark
  regions (hall-cam doorway, corrupted Buttons sun).
- **`drawScareFace(t)`** — the death face: rolling eye-whites in hollow
  sockets, two rows of teeth, creeping veins, slice displacement + vertical
  screen-melt. Also flashed for 1–2 frames as a subliminal (`sublimT`) and
  inside the studio-intro corruption.

## MOSH studio intro

`ST.STUDIO`, driven by `drawStudio(stateT)`; total length `STUDIO_LEN`
(9.6s), skippable after 1.5s. Phases:

1. **0–1.5s** tape load: VCR OSD, tracking tears, "MOSH VIDEO" blue slate.
2. **1.5–5s** logo build: chrome-gradient letters M-O-S-H fly in with ghost
   trails (landing times in `LAND`), shockwave rings, rotating sunrays,
   orbiting sparkles, lens-flare sweep at 4.2s.
3. **5–7.2s** ident: "MOSH PRODUCTION STUDIO · PRESENTS" + gold glow pulse.
4. **7.2–9.6s** corruption: chromatic-aberration split, vertical melt
   slices, palette rots to green (`chromeGrad(decay)`), audio detunes down,
   2-frame scare-face flash at 8.45s, static wipe → MENU.

Keep visual phase timings in sync with `studioAudio()` — both hardcode the
same timeline constants.

## Testing / debugging

- Serve with `python3 -m http.server 8000`; needs one click for audio.
- `window.__dbg` is exposed: `__dbg.state`, `__dbg.min`, `__dbg.ff(minutes)`
  to time-travel (e.g. `__dbg.ff(214)` right before the evil call),
  `__dbg.threat`, `__dbg.integrity`.
- Headless check (Playwright): use the preinstalled Chromium at
  `/opt/pw-browsers/chromium` if in a Claude remote env; click the page to
  pass BOOT, then drive via `__dbg`.

## Known trade-offs / ideas not yet done

- Anomalies freeze during the 13s EAS state (acceptable; phone does not
  pause gameplay anymore).
- Roadmap (from README): multiple nights, "the game remembers your name"
  via localStorage, real-system-time timestamp leaks, hidden ending for
  never watching channel 6, Steam wrap via Tauri/Electron.
