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
  emulation, gated behind explicit driver arming and a two-tier safety design: brake press
  is the only thing that latches the whole session off until the next cruise re-engagement
  (gas/steering-torque no longer do — a deliberate, driven, tuned call), but gas, brake,
  *and* steering-torque still independently block sending a simulated button press in the
  exact instant any of them is active, every cycle, regardless of the session latch.

## How it actually works (the short version)

Someone asked this exact question on
[sunnypilot's own forum](https://community.sunnypilot.ai/t/enabling-icbm-on-17-impreza/191)
and got a real, honest answer from sunnypilot itself:

> "The necessary button message structure and routing aren't exposed or implemented by
> Subaru in a way that supports emulation yet... this isn't something we're actively
> pursuing... would likely need to be a community-driven effort."

That's true of the message everyone assumes is the mechanism — the visible steering-wheel
button bits on `CruiseControl` (`0x144`). Those really are blocked: not TX-allowed by the
panda's safety firmware, and even if they were, EyeSight's own copy is already live on that
bus, so injecting there means fighting a real, active transmitter.

The actual answer was a different message. `ES_Distance` (`0x161`) carries its own
`Cruise_Button` field — and openpilot was *already* rebuilding and re-transmitting that
entire message, every single frame, on every drive, for reasons that have nothing to do
with buttons. Three things about it made emulation possible with zero new hardware, zero
firmware changes, zero new bus access:

1. **It's already being rewritten, every frame.** The button field just needed to carry a
   chosen value instead of a relayed one — no new code path, no new message.
2. **It's already TX-allowed.** No panda safety-firmware change required.
3. **Nothing else is fighting for the wire.** The panda's own relay blocks the camera's
   copy of this exact message from ever reaching the main bus — openpilot is the sole
   legitimate source, zero contention. That's the opposite of `0x144`, where a live
   competing transmitter is exactly why that route is blocked.

Confirmed live, not just on paper: a software-chosen button value engaged real cruise
control on the actual car within ~100ms, matching the car's exact speed at that instant.
Full derivation and the live test: `progress.md` (search Q4/Q6/Q10), and
`research/es_distance_cruise_button_finding.md`. Sunnypilot calls this exact idea —
button-spoofed longitudinal control — **Intelligent Cruise Button Management**, confirmed
unimplemented for every Subaru platform until now. Hence the name.

## Not install-ready for a stranger yet — the honest gap list

If you're just here to read the research, skip this. If you're thinking about installing
the actual fork, know this first:

- **The curve and lead advisories can't be turned off, for anyone, including you.** This
  branch ships prebuilt (no on-device rebuild is possible), and two features' settings
  keys were never compiled into the params allowlist — they silently fall back to
  permanently *on*, with no UI toggle and no working SSH override. Fixing this needs a
  real reinstall, not a config change.
- **The map-data trap that cost this project weeks to diagnose has no safeguard added.**
  If your device has no downloaded map data for your region, the curve advisory arms
  correctly, shows nothing is wrong, and simply never fires — silently, indefinitely, with
  zero on-screen indication. Check your region is actually downloaded before assuming it's
  broken or working.
- **Real actuation (the button-spoofing part) is SSH-only, on purpose for now** — three
  flag files, no on-device toggle. Given the safety stakes, that's arguably correct to
  leave as a deliberate gate rather than a gap, but it does mean a true stranger can't
  reach it without dev-level access either way.
- **The tuned constants (decel rate, button cadence, EyeSight's floor) came from one car.**
  Actuation isn't restricted beyond "any preglobal Subaru" — it'll run on other model
  years using numbers only validated on this one.
- No first-boot disclaimer exists anywhere in this branch. You will not be told any of
  this by the software itself.

Full trace: `progress.md`, 2026-08-14 entries.

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

## How this got here

Every step followed the same discipline: verify against real telemetry or source before
relying on anything, and revert immediately if a live test misbehaves rather than debug
live in a moving car. That discipline is also what caught this project's own mistakes,
including two early AI-research passes that turned out to contain fabricated quotes
(excised, documented, never repeated — see `claude.md`).

**[`research/README.md`](research/README.md)** maps all ~80 files in that directory —
a flowchart of how each finding led to the next, and which results are confirmed vs. still
open vs. dead ends — so you don't have to open each one to find out. See
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
