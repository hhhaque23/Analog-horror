# DEAD AIR — an analog horror game

**WKRX-TV, Grayson County. 12:00 AM.** You are the overnight master-control
operator. Six channels. One rule from the 1987 handbook: *the transmitter must
never carry an uncorrected signal.* Something travels through broadcast
infrastructure, and tonight it found your station.

Everything — visuals, VHS distortion, static, the 60 Hz hum, the EAS attention
tone, the stingers — is generated procedurally at runtime. No asset files, no
dependencies, no build step. One HTML file.

## Play it

Open `index.html` in any modern browser (click once to enable audio), or:

```bash
python3 -m http.server 8000
# then visit http://localhost:8000
```

**Headphones, lights off, fullscreen (F11).** Contains sudden loud sounds and
flashing imagery.

## How to survive the shift

| Input | Action |
|---|---|
| `1`–`6` | Switch channel |
| `SPACE` (hold) | Re-track a corrupted channel until the signal is restored |
| `E` | Answer the phone, if it rings |

- Intrusions corrupt channels at random. Find them, tune in, hold SPACE.
- Ignore one too long and it becomes a **containment failure**. Three failures
  and it reaches master control.
- The shift runs 12:00 AM → 6:00 AM (one game minute per real second — about a
  six-minute night, escalating hard after 4 AM).
- Channel 6 is dead static. The last operator is still on it.

## Shipping to Steam

The game is a static web page, so wrap it in a desktop shell:

1. **Tauri** (recommended, ~3 MB binaries): `npm create tauri-app`, point it at
   this directory as the frontend, build for Windows/macOS/Linux.
2. **Electron + electron-builder** if you prefer the larger but more familiar
   toolchain.
3. Integrate `steamworks.js` (Electron) or the Steamworks Rust crate (Tauri)
   for achievements ("Answered the Phone", "Zero Failures", "Watched Channel 6
   for a Full Minute").
4. Steam Direct fee is $100 per title; you'll need store capsule art, a
   trailer, and content warnings for flashing lights and sudden audio.

## Roadmap ideas

- Multiple nights with a tape-archive framing (Night 2: the phone calls back).
- Persistent "the game remembers your name" scares via `localStorage`.
- VHS timestamp glitches that leak real system time.
- A hidden ending for operators who never once look at channel 6.
