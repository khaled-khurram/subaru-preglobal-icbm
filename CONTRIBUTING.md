# Contributing

This project started as one person's research log on a single 2015 Subaru Outback. It's
public because the findings generalize to every preglobal Subaru with EyeSight (2015-2019,
Forester/Impreza/Legacy/Outback/WRX), and because a lot of the open questions need more
data — more cars, more model years, more drives — than one car can provide.

## The ground rule, before anything else

**Verify against real telemetry, real source code, or a directly-quoted first-hand
source — never assumption, and never a fabricated citation.** This project was burned
twice early on by AI-assisted research passes that invented "verbatim quotes" that didn't
exist in the sources they claimed to cite (see `claude.md` and the changelog in
`progress.md` for the full story of catching and excising that). Every claim that survived
into the current docs was independently re-verified. Keep it that way. If you're not sure
whether a claim is solid, say so explicitly rather than smoothing it over — this repo's
docs are full of "AMBIGUOUS," "INCONCLUSIVE," and "unverified" labels on purpose, and that
honesty is more valuable than a clean-looking result.

If a live test starts misbehaving, **revert immediately and investigate after** — don't
debug live in a moving car. That discipline has caught real bugs before they became real
problems, more than once.

## What's actually useful right now

- **You own a preglobal Subaru (any year, 2015-2019) with EyeSight.** The biggest gap in
  the current research is that it's all one car. If you have an archived route/CAN log
  (or are willing to capture one passively — nothing here requires transmitting anything),
  running the existing analysis scripts (`research/*.py`) against your own data and
  reporting whether the findings hold is extremely valuable, and it's zero-risk.
- **Passive analysis and archive mining.** Most of the real findings in this repo (button
  bit positions, the `Cruise_Throttle` command-channel hypothesis, cadence limits) came
  from correlating already-logged CAN data before anything was ever transmitted. That
  pattern — pre-register what would confirm or kill a hypothesis, then check the archive —
  is the highest-value, lowest-risk way to contribute.
- **Source/documentation review.** Read a research doc, check its claims against
  `opendbc`/`openpilot`/`sunnypilot` source or the Discord archive, and open an issue if
  something doesn't check out.
- **Live testing, if you're comfortable with the risk.** See Safety below. Anything
  proposed for live actuation should already have a written design and a passive/bench
  validation step behind it — check `research/phase3_controller_design.md` for the shape
  this project expects that to take.

## How to actually contribute

- **Issues** are the easiest entry point — a bug report, "does this hold on my car,"
  or a question, no code required.
- **Pull requests**: fork the repo, make your changes on a branch, open a PR against
  `main`. Direct pushes to `main` aren't accepted from anyone, including the maintainer —
  everything goes through review, so the process is the same for a first-time contributor
  as it is for the person who started this.
- For anything that touches `research/*.py` or proposes a new live test: state your
  hypothesis and pre-registered pass/fail criteria *before* showing results, the same way
  the existing scripts do. Post-hoc reinterpretation of a result is the one thing this
  project has explicitly flagged as a trap to avoid (see the round-1/round-2 stratification
  notes in `research/es_stage0_followup_results.md` for a worked example of doing this
  right, including a result that was later found to need discarding — recorded as such
  rather than quietly dropped).

## Safety

This repository documents modifying how a vehicle's adaptive cruise control behaves,
including CAN message injection and closed-loop longitudinal actuation. That's inherently
safety-adjacent, not a toy project. A few real incidents are documented in `progress.md`
exactly as they happened — a bus-injection attempt that faulted EyeSight off, a UI change
that caused real highway disengages — because hiding failures would defeat the point of
keeping a rigorous log.

If you're proposing or running a live test:
- Passive/archive validation first, always, before anything transmits.
- Bench or parked testing before a moving vehicle.
- Empty road, low stakes, before normal driving conditions.
- A real, working override/kill mechanism, verified before you need it.
- Assume nothing about how your specific car's model year or trim will behave — verify
  it, don't extrapolate from someone else's.

Nothing in this repository is a certified or warranted product. See `LICENSE`.
