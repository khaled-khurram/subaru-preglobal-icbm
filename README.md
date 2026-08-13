# subaru-preglobal-icbm

**ICBM for preglobal Subarus — real, driven on public roads, and far from finished.** This
project rides EyeSight's own ACC via steering-wheel button emulation, on a 2015 Outback
running [sunnypilot](https://github.com/sunnypilot/sunnypilot). It works. It also has real,
unsolved problems — that's the actual reason this repo is public. See
[Known issues](#known-issues--this-is-not-a-finished-product) below, and help fix them.

**Start here:** [`progress.md`](progress.md) — the full log, exactly as it happened,
failures included.

## What works

- **Curve-speed advisory** — map-based curve detection (MTSC), driver-facing alert.
- **Lead-vehicle closing-speed advisory** — same idea, driven by the vision model's own
  lead-detection data.
- **Closed-loop ICBM actuation** for both — EyeSight's own setpoint commanded via button
  emulation, gated behind explicit driver arming and an instant override latch on brake
  press (gas/steering-torque overrides are disabled in the current live config — brake-only
  is what's actually tuned and driven).

## Known issues — this is not a finished product

- **EyeSight's own braking is slow.** Self-measured on this car: ~1.94 mph/s. One data
  point from one car, not rigorously validated — worth re-measuring on other cars.
- **Doesn't coordinate with EyeSight's own lead-lock.** If EyeSight already has the car
  ahead locked and is slowing on its own, this system can still layer an unnecessary
  advisory/adjustment on top of it instead of recognizing EyeSight already has it handled.
- **Doesn't always restore speed once the event clears.** E.g. after a curve, back on a
  straight — sometimes it just stays at the reduced speed instead of walking back up.
- **Restore often lands ~1mph under the original set speed** (75 → 70 → 74, not 75) instead
  of exactly recovering it. Plausibly related to a documented ECU debounce quirk on
  closely-spaced button presses (`progress.md`, Q10's burst finding), not yet confirmed as
  the actual cause.
- **Inconsistent on speed-limit / city / construction-zone adjustments** — auto-adjusts
  some of the time, not reliably.
- **Only 1mph "shallow" steps are implemented.** The car supports ±5mph "deep" steps too,
  but the signal needed to confirm they'd behave safely turned out unreliable in passive
  testing — needs a dedicated live test before it ships.
- **The curve advisory can suggest a speed EyeSight physically can't reach** — e.g. 22mph
  for a tight roundabout, below EyeSight's real ~25mph floor. Actuation is clamped to that
  floor; the displayed advisory number isn't guaranteed to be.

## Known bugs in this project's own code

- A past UI feature (on-screen status dots) is directly implicated in repeated real "TAKE
  CONTROL" disengages on a 3-hour highway drive — fully reverted same night. ~90%
  confidence the code caused it; the exact mechanism was never fully pinned down. Full
  writeup in `progress.md`.
- Separately, and less severe: occasional (roughly weekly), brief (2-3 second)
  self-resolving "TAKE CONTROL" flashes with no actual loss of control — car keeps driving
  normally throughout. Not yet root-caused; may or may not be related to the above.

## Ideas worth building

- **Deep-step (±5mph) actuation** — faster large corrections. Needs the live magnitude
  test above before it's safe to wire up.
- **Follow-distance emulation** — temporarily widen the gap setting to let a car merging
  onto the highway in, then restore it. A genuinely new idea; whether the follow-distance
  button is even CAN-commandable on this platform hasn't been researched yet.
- Have a bug, an idea, or a different preglobal car to test on? Open an issue. That's the
  whole ask.

## The longer-term question

Full longitudinal control — replacing EyeSight outright, not riding it — has never been
solved for any preglobal Subaru, in almost a decade of community attempts. This repo is
investigating that too, but it is explicitly *not* the near-term focus; perfecting ICBM
comes first. See `research/eyesight_throttle_channel.md` and Q14 in `progress.md` if you
want the details.

## Why "ICBM"

Sunnypilot already ships this exact idea — button-spoofed longitudinal control — for other
car brands, under the name **Intelligent Cruise Button Management**. It's confirmed
unimplemented for every Subaru platform, preglobal included. This project is an attempt to
close that gap, starting from a single car, done properly.

## How this got here

Every step followed the same discipline: verify against real telemetry or source before
relying on anything, and revert immediately if a live test misbehaves rather than debug
live in a moving car. That discipline is also what caught this project's own mistakes,
including two early AI-research passes that turned out to contain fabricated quotes
(excised, documented, never repeated — see `claude.md`).

See [`research/`](research) for every individual write-up, and
[`CONTRIBUTING.md`](CONTRIBUTING.md) for how to help — right now, the most useful thing is
running the existing passive analysis scripts against a different preglobal car's own
archived data.

## Safety

This repository documents CAN-level changes to how a real vehicle's adaptive cruise
control behaves. That's safety-adjacent by nature, not a toy project — read
[`CONTRIBUTING.md`](CONTRIBUTING.md#safety) before proposing or running anything live.
Real incidents, including ones that faulted the car's own safety systems, are documented
in `progress.md` exactly as they happened.

## A note on paths

A couple of the research scripts/docs reference a local route-archive location and
schema path from this project's own setup — those have been scrubbed before publishing.
Swap in your own paths if you want to actually run them.

## License

[MIT](LICENSE) — matching the license of `openpilot`, `opendbc`, and `sunnypilot`
underneath it. Nothing here is a certified or warranted product; see the license's
additional notice and the Safety section above.
