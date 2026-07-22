# HANDOFF — Claude context for DEAD AIR

Read this first if you are an AI assistant (or a human) picking up this repo.

## What this is

**DEAD AIR** is a single-file analog horror game with an FNAF-style survival
loop. You play the overnight master-control operator at WKRX-TV: watch six
channels, purge signal intrusions with a manual-tracking minigame, manage a
finite power supply, and keep a physical entity from reaching your booth,
from 12:00 AM to 6:00 AM. Everything — rendering, VHS effects, monsters,
music, sound design — is generated procedurally at runtime.

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

Sections top-to-bottom (search the banner comments):

1. **utility** — `rnd/ri/clamp/lerp/pick`.
2. **audio** — `initAudio()` builds persistent beds: 60 Hz mains hum,
   looping noise static (`setStaticLevel`), dread drone (`setDroneLevel`),
   whisper bed (`setWhisperLevel`). One-shots: `beep`, `stinger`, `scream`,
   `screech`, `thump` (heartbeat), `footstep`, `knock`, `advanceCue`
   (entity moved), `floodBlast`, `easTone`, `phoneRingTone`,
   `evilRingTone` (slow/low = the tell), `silenceScare`.
3. **MOSH studio intro audio** — `studioAudio()` / `stopStudioAudio()`.
4. **game state** — `ST` enum:
   `BOOT → STUDIO → MENU → INTRO → PLAY ⇄ EAS → SCARE → DEAD | WIN`,
   plus `BLACKOUT` (power death sequence). Phone is an overlay in PLAY,
   never a state.
5. **input** — `advance()` any-key dispatcher; PLAY keys: `1–6` channels,
   `SPACE` toggle tuner, `←/→` or `A/D` steer, `F` signal flood, `E` phone.
6. **anomaly system + tuner** — core loop, below.
7. **entity system** — `entityTick`, `signalFlood`, below.
8. **power** — `powerTick`, below.
9. **scripted events** — `buildEvents()` timeline + `HOURLINES` + `CALLS`.
10. **drawing helpers** — `drawStatic`, `text`, `fmtClock`, `corruptStr`,
    `trackingLines`.
11. **monsters** — `drawFigure` (the Tall One), `drawCrawler`,
    `drawWatcher`, `drawScareFace`.
12. **MOSH studio intro visuals** — `drawStudio(t)` + `drawMoshLogo`.
13. **channel renderers** — `chTestCard/chWeather/chCamera/chButtons`,
    dispatched from `drawPlayChannel` (which also draws the door-bleed
    overlay and brown-out dimming).
14. **tuner overlay** — `drawTuner()`.
15. **screens** — boot/menu/intro/EAS/dead/win + BLACKOUT case in `frame()`.
16. **main loop** — `frame()`; PLAY runs `anomalyTick → entityTick →
    powerTick → scriptTick`, each guarded by a state check (any can kill).

## The three pressure systems (FNAF core)

### 1. POWER (`power`, `powerTick`)

- Starts 100. Passive drain 0.24 %/s (≈14 % margin over the 360 s night).
- Costs: tuner active +1.5 %/s · SIGNAL FLOOD −10 flat · answering phone −2.
- UI: `#powerbar` DOM element `PWR ▮▮▮… 62%`, red `.low` class under 25 %.
  Below 40: canvas dims (overlay in `drawPlayChannel`) and master gain sags.
- At 0: `ST.BLACKOUT` — dark screen, footsteps, eyes fading in, jumpscare
  death after 8.5 s. **Dawn clause:** if `gameMin ≥ 348` when power dies
  (or 6 AM arrives mid-blackout), you survive with integrity capped at 60.

### 2. The entity walks (`entityPos`, `entityTick`)

Track: `0 LOT → 1 HALL → 2 DOOR → 3 = death`. Advance timer
`rnd(45,70) * entityAggro()` where `entityAggro() = lerp(1,.38,hour/5) *
0.8^strikes`. Each advance plays `advanceCue` (louder per stage).

- **Cameras are the information loop:** at LOT it renders on ch3 (cam A),
  at HALL on ch4 (cam B, creeping toward the lens as its timer runs down).
  At DOOR the cameras show *empty rooms* — that absence is the warning.
- **At DOOR:** knocking, fingers + an eye around the right frame edge on
  every channel, red alarm border, countdown. Window `rnd(7,9)` s
  (`rnd(4,6)` after 3 AM). No flood in time → death
  ("IT CAME THROUGH THE DOOR").
- **`F` SIGNAL FLOOD** (`signalFlood`): −10 power, knocks it back
  `ri(1,2)` stages — but only if it is at HALL/DOOR; flooding an empty lot
  wastes the power (punishes panic).

### 3. Anomalies + the tuner minigame (`anomalyTick`, `tuneTick`, `drawTuner`)

`anomalies[ch] = {age, kind, resp, fuse, lockNeed, warned}`.

| resp | kinds | Cleared by | Fuse | lockNeed |
|---|---|---|---|---|
| `track` | figure/face/text/invert | tuner minigame | 32 s | 1.8 s in-band |
| `track` | `surge` | tuner (wider band, violent jolts) | **10 s** | 1.0 s |
| `stare` | `stare` | **look away** (grows 2.6×/s watched, decays .85×/s ignored) | 26 s | — |

Tuner: SPACE toggles; a needle (drift + jolt impulses, damped velocity)
must be steered into a relocating sweet-spot band. In-band fills `lock`;
out-of-band decays it + `screech()`. At 50 %/85 % lock the anomaly
"lunges" (stinger blip + shake). After 3 AM random 0.5–0.9 s **input
inversion** windows (UI flags "►◄ INVERTED"). Fuse keeps ticking while
tuning. Expiry = strike (−12 integrity); **3 strikes = death**.

### Threat / dread

`threat = clamp(hour/6*.42 + strikes*.18 + anomCount*.06 + ch6Heat + dread, 0, 1)`
drives drone/hum volume, grain, OSD glyph corruption (`corruptStr`),
subliminal 1-frame face flashes (>.5), heartbeat (>.55). `dread` comes from
bad choices and decays.

### Channel 6

Watching accrues `ch6WatchT`: whispers, `ch6Heat`, and after 3 s the static
names `nextTarget` (the pre-chosen next anomaly channel). The dead operator
stands in the static after ~1:30 and turns to face you
(`opFacing = clamp((gameMin-120)/240,0,1)` → `drawFigure` `facing` param).

### Scripted timeline (gameMin; 1 real s = 1 min)

45 hum shift · 60 EAS · 95 call 0 (friendly) · 150 call 1 · 185 Buttons
corrupts · 215 **call 2 = the entity** (evil ring: slow/low; answering
spawns 2 anomalies, +dread, −8 integrity, −2 power; ignoring +4 integrity)
· 250 triple intrusion · 290 silence scare · 330 dread bump · 360 WIN.
Hourly chimes + `HOURLINES` between.

### Scoring

`integrity` 100 start: −12/strike, −.12/s per active anomaly, +2 restore.
Final score = `integrity + power*.1` → S ≥95, A ≥82, B ≥66, C ≥46, else D;
any death = F, with `deathCause` shown on the DEAD screen. Last grade in
`localStorage["deadair_last"]`, shown on the menu.

## The monsters

- **`drawFigure(x,y,scale,dark,facing)`** — the Tall One: teleport-twitch
  stillness, threat-scaled head tilt, eyes tracking screen center,
  splayed fingers, breathing scale. Also the door-burst jumpscare
  (SCARE state first .35 s) and the win-screen cameo.
- **`drawCrawler`** — stop-motion (130 ms quantized) all-fours thing on
  the cameras after 2 AM.
- **`drawWatcher`** — blinking eye-pairs in dark regions.
- **`drawScareFace(t)`** — death face: rolling eye-whites, two teeth rows,
  veins, slice displacement + vertical melt; also the subliminal flash and
  the studio-intro corruption frame.

## MOSH studio intro

`ST.STUDIO`, `drawStudio(stateT)`, `STUDIO_LEN` 9.6 s, skippable after
1.5 s. Phases: tape-load (0–1.5) → chrome letters fly in (`LAND` times,
sunrays/sparkles/flare) → ident glow (5–7.2) → corruption (chromatic
split, melt, 2-frame scare face, static wipe). `studioAudio()` hardcodes
the same timeline — keep them in sync.

## Testing / debugging

- Serve: `python3 -m http.server 8123`; one click needed for audio.
- Headless: Playwright + `/opt/pw-browsers/chromium`
  (`NODE_PATH=/opt/node22/lib/node_modules` in Claude remote env).
- `window.__dbg`: `state`, `min`, `ff(m)` (time-jump that marks skipped
  events done), `threat`, `integrity`, `power` (get/set), `entityPos`
  (get/set — setting 2 arms the door), `doorT`, `tuneOn`, `lock`,
  `needle`, `band`, `anomalies`, `spawn()`, `cause`.
- Deploy: Railway serves the repo **default branch**
  (`claude/analog-horror-game-jzju84`); `main` is kept in sync. Dev happens
  on `claude/game-atmosphere-intro-64q3fc`, then fast-forward both.

## Known trade-offs / ideas not yet done

- Anomalies/entity freeze during the 13 s EAS state (accepted).
- Roadmap: selectable difficulty nights, "the game remembers your name",
  hidden ending for never watching ch6, Steam wrap via Tauri/Electron.
