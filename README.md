# DEAD AIR — an analog horror game

*a MOSH Production Studio release*

**WKRX-TV, Grayson County. 12:00 AM.** You are the overnight master-control
operator. Six channels. One rule from the 1987 handbook: *the transmitter must
never carry an uncorrected signal.* Something travels through broadcast
infrastructure, and tonight it found your station.

Everything — the animated MOSH studio ident, visuals, VHS distortion, static,
the 60 Hz hum, the EAS attention tone, the whispers, the stingers — is
generated procedurally at runtime. No asset files, no dependencies, no build
step. One HTML file.

> Contributing or continuing development with an AI assistant? Read
> [`HANDOFF.md`](HANDOFF.md) first — it maps the whole codebase.

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
| `SPACE` | Engage/abort manual tracking on a corrupted channel |
| `←` `→` (or `A`/`D`) | Steer the tracking needle into the sweet-spot band |
| `F` | **SIGNAL FLOOD** — repels the thing in the hallways. Costs 10% power |
| `E` | Answer the phone, if it rings — *or don't* |

- Intrusions corrupt channels at random. Tune in, hit SPACE, and **steer the
  needle** into the moving band until LOCK hits 100% — while the thing in the
  signal lunges at you and, late at night, inverts your controls.
- **Surges** give you only ten seconds, but lock fast if you keep your nerve.
- Some intrusions **watch back**. Those grow while you look at them; cut away
  and starve them.
- Miss a fuse and it's a **containment failure**. Three failures and it
  reaches master control.
- **It doesn't only live in the signal.** It crosses the parking lot (cam A),
  then the hallway (cam B), then it is at your door. When the cameras go
  empty, FLOOD — or it comes through.
- **Power is finite.** Everything costs: tracking, flooding, even the phone.
  Run dry before dawn and you'll hear footsteps in the dark.
- The phone rings three times a night. Not every caller is staff — listen to
  the ring before you press E.
- Channel 6 is dead static. The last operator is still on it. Stare long
  enough and the static names the next channel to fail… but the static also
  notices you.
- The shift runs 12:00 AM → 6:00 AM (one game minute per real second — about a
  six-minute night). It is meant to be hard: expect to die before your first
  sunrise. Grades S–D on integrity + leftover power; death is an F. Your last
  grade is remembered.

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
