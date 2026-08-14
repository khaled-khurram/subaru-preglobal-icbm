# Installing this

Written for someone who has a comma device and a supported car, and hasn't touched this
project before. If a step below doesn't work exactly as written, that's useful to know —
tell us, don't just work around it.

**Read the "Not install-ready for a stranger yet" section of [`README.md`](README.md)
first.** This isn't a finished product. Two alerts currently can't be turned off (fix
planned, see `research/phase3_install_readiness_plan.md`), and the real actuation feature
is intentionally locked behind developer-level access — not a bug, a safety decision.

## What you need

- A comma device, set up and working with a supported car harness for your Subaru.
- A **2015–2019 Subaru with EyeSight, preglobal generation** (Forester, Impreza, Legacy,
  Outback, WRX). If you're not sure which generation your car is, check
  [opendbc's car list](https://github.com/commaai/opendbc) before continuing — installing
  on the wrong platform won't work and isn't worth troubleshooting blind.
- Comfort with SSH if you ever want to change anything beyond the defaults. Nothing here
  requires it to just drive with the defaults on.

## Installing

1. On the comma device's setup screen, choose "Custom Software" (exact wording may vary
   by device software version).
2. Enter this install address: `install.sunnypilot.ai/fork/<github-username>/phase3-actuation`
   — **placeholder note: confirm the exact live username before using this** (the
   maintainer's GitHub account is being renamed as of this writing; this file will be
   updated once that's settled, but double-check rather than assume it's current).
3. Let it install and reboot. First boot may take a few minutes.
4. If you're installing over an *existing* different fork or the official sunnypilot
   release: this is a full factory reset — you'll need to redo camera/steering
   calibration afterward. Switching *branches* on an already-installed copy of this same
   fork does not require that.

## What happens automatically — no setup needed

- **Curve-speed advisory**: a "Curve Ahead" alert with a suggested speed, on curves at
  35+ mph. On by default, currently no way to turn off (see the gap list in `README.md`).
- **Lead-vehicle closing advisory**: a "Traffic Ahead" alert if a car ahead is closing in
  fast and you haven't reacted yet. On by default, same caveat.

Both are advisory-only — they tell you something, they don't touch the car. You still
drive normally.

## What does NOT happen automatically, on purpose

**Real automatic actuation — the system actually adjusting EyeSight's cruise-control
setpoint for you — is intentionally locked behind SSH access.** This isn't an oversight.
It changes the car's actual speed without your foot on a pedal, and this project doesn't
think that should be one tap away for someone who hasn't read the safety notes. If you
want it, read `progress.md` and `research/phase3_controller_design.md` first, then ask.
This file isn't the place to learn how to arm it.

## How to turn the advisories off (once the fix in `research/phase3_install_readiness_plan.md` ships — not yet true today)

SSH into the device and run:

```
touch /data/curve_advisory_disabled       # turns off the curve alert
touch /data/lead_closing_advisory_disabled  # turns off the lead-closing alert
```

Delete either file to turn it back on. Takes effect within a few seconds, no reboot.

## How do I know it worked?

Drive somewhere with a curve you know is sharp enough to normally need to slow down for.
You should see a "Curve Ahead" alert with a distance and suggested speed before you reach
it. If you never see one anywhere, something's wrong — check the "Known issues" list in
`README.md` first (the map-data requirement is a real, documented trap: if your device
hasn't downloaded map data for your area, this will silently do nothing forever with no
warning).

## Uninstalling / going back to normal

- **Going back to official sunnypilot or a different fork**: use the normal install
  process for whatever you're switching to. This is a full factory reset either way —
  same cost as installing this was.
- **Just switching away from this branch, staying on this same fork**: doesn't need a
  reset, but currently leaves a few small files behind on the device
  (`/data/phase3_*`) that do nothing under other branches — harmless clutter, not
  automatically cleaned up yet.

## Something's wrong / you have an idea

Open an issue on the repo, or see `CONTRIBUTING.md`. Tell us exactly which step above
didn't match what you saw — that's the most useful bug report this project can get right
now.
