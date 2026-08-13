# subaru-preglobal-icbm

Real longitudinal control for preglobal Subarus (2015-2019 Forester / Impreza / Legacy /
Outback / WRX with EyeSight) is a solved problem for exactly zero cars, and hasn't moved in
years — every official effort stopped at the 2020+ "global" platform, and every community
attempt at preglobal has died at the same wall: EyeSight won't let go of the bus.

This is a from-scratch research project chasing that wall from a different angle — riding
EyeSight's own decisions instead of fighting it — built and documented on a real 2015
Outback running [sunnypilot](https://github.com/sunnypilot/sunnypilot), with one rule
enforced the whole way: **verify every real claim against actual telemetry, real source
code, or a directly-quoted first-hand source before relying on it.**

**Start here:** [`progress.md`](progress.md) — the full living log. Every phase, every
open question, every decision, every incident, exactly as it happened, including the ones
that didn't work.

## What's actually shipped and live-tested

- **Curve-speed advisory** — map-based curve detection (MTSC), wired to a driver-facing
  alert, live and working.
- **Lead-vehicle closing-speed advisory** — a second advisory trigger, grounded in real
  telemetry analysis of the vision model's own (previously unused) lead-detection data.
- **ICBM-style closed-loop actuation** — on a platform with no native long-control support,
  the car's own ACC (EyeSight) is commanded via steering-wheel button emulation: not
  replacing EyeSight, riding its own setpoint. A closed-loop controller for curve-speed and
  lead-vehicle-closing scenarios, gated behind explicit driver-controlled arming and an
  unconditional, session-long override latch — brake, gas, or steering torque instantly
  and permanently disables it. Deployed and live-tested on public roads. **This is real,
  proven, and working today** — full openpilot-style longitudinal control (replacing
  EyeSight outright) is not, see below.

## What's currently under investigation — and the honest state of it

Every actuation above works through one field of one CAN message: the cruise button. The
same message also carries a continuous throttle-command field that openpilot already
transmits, unedited, on every drive — a possible second command channel this project may
already own. Source-verified against upstream `opendbc` and sunnypilot's own fork,
cross-checked against first-hand community testimony recovered from a decade of Discord
history, then put through a full falsification-first passive archive campaign before
anything was ever transmitted.

**The result so far is a genuine, reported-as-is inversion of the original premise, not a
confirmation.** Of the three candidate command fields, the one with the cleanest evidence
that it's a real, obeyed command (`ES_Brake.Brake_Pressure`) is the one that needs a panda
safety-firmware change before it could ever be written. The one field writable today with
zero firmware change (`Cruise_Throttle` — the whole reason this looked cheap) came back
inconclusive, not confirmed. **Full longitudinal control is not proven and not close to
being deployed** — this is still archive analysis, not a live result. See
`research/eyesight_throttle_channel.md`, `research/es_stage0_campaign_synthesis.md`, and
Q14 in `progress.md` for the full case, including where it currently stands short.

## Why "ICBM"

Sunnypilot already ships this exact idea — button-spoofed longitudinal control — for other
car brands, under the name **Intelligent Cruise Button Management**. It's confirmed
unimplemented for every Subaru platform, preglobal included. This project is an attempt to
close that gap, starting from a single car, done properly.

## How this got here

Every step — CAN reverse-engineering, UDS diagnostic probing, bus-topology analysis, panda
safety-firmware research, community archaeology across a decade of the comma.ai Discord —
followed the same discipline: verify before relying on anything, pre-register what would
confirm or kill a hypothesis before looking at the result, and revert immediately if a live
test starts misbehaving rather than debug live in a moving car. That discipline has caught
real bugs before they became real problems — a `plannerd` crash from a schema mismatch, a
safety-latch false-positive from engagement timing, an under-sized safety budget — and has
caught this project's own mistakes just as readily, including two AI-research passes that
turned out to contain fabricated quotes (excised, documented, never repeated).

See [`research/`](research) for every individual write-up, and
[`CONTRIBUTING.md`](CONTRIBUTING.md) if you want to help — the single most useful thing
right now is running the existing passive analysis scripts against a different preglobal
car's own archived data.

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
