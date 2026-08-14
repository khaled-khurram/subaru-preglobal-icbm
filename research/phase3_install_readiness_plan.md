# Install-readiness plan: fixing the always-on alerts without touching the UI

**Status: design only, not implemented, not pushed.** Written 2026-08-14, multi-session
scope — the intent is to review this over more than one sitting, not rush it into the
daily driver. Nothing in this doc has been applied to `khaled-khurram/sunnypilot` yet.

## Hard constraint, stated first because it shapes everything below

**No changes to `selfdrive/ui/` or any rendering/onroad code, full stop.** The 2026-07-26
status-dots incident (`progress.md`, Q13) put real "TAKE CONTROL IMMEDIATELY" disengages
on a live highway drive, at ~90% confidence traced to that UI code. The commit currently
running is clean of that entirely, and it stays that way. Every fix below is pure
control-loop Python — nothing that touches what renders on screen.

## The problem, restated precisely

Confirmed directly against the compiled binary, not inferred (`progress.md`, 2026-08-14):
`CurveSpeedAdvisory` and `LeadClosingAdvisory` are permanently **on** for every installer
of this branch, with **no way to turn either off** — no on-device toggle exists for
either, and SSH can't fix it either, because `Params().put_bool()` checks the same
compiled allowlist that `get_bool()` does and raises the identical `UnknownKeyName`.
`LeadClosingTestGuidance` has the mirror problem: permanently **off**, and its own code
comment documents an SSH opt-in (`Params().put_bool("LeadClosingTestGuidance", True)`)
that cannot work for the same reason.

Root cause: this branch ships prebuilt (no `SConstruct` anywhere in the tree — confirmed,
not assumed), so the compiled `common/params_pyx.so` can never be rebuilt on-device.
Adding a key to `common/params_keys.h`'s *source* does nothing without a real reinstall
that goes through sunnypilot's actual build pipeline — which isn't happening any time soon
(this is a multi-session, deliberately unhurried fix).

## The fix: reuse the flag-file pattern already proven safe in this exact codebase

Phase 3 actuation arming already solved this identical problem — "new on/off state, no
compiled-key access, no UI" — with a plain file-existence check
(`sunnypilot/selfdrive/controls/lib/phase3_shared.py`):

```python
def is_armed(flag_file: str) -> bool:
  return os.path.exists(flag_file)
```

No `Params()`, no compiled allowlist, no rebuild, no UI. It's been live and safe since
Phase 3 first shipped. Apply the same pattern to the two broken advisory toggles — but
**inverted**, since the goal is to preserve current behavior exactly, not change it:

- **Today's actual behavior, unconditionally: both advisories are on.** The fix must not
  change that for the car this is already running on. So the new mechanism is an
  **opt-out** file, not an opt-in one — *absence* of the file means "on" (identical to
  today), *presence* means "off."
- `LeadClosingTestGuidance`'s existing default (off) is preserved the same way, opt-in
  file this time, since that one is already correctly off by default and just needs its
  broken SSH instruction replaced with one that actually works.

### Exact proposed change

`sunnypilot/selfdrive/controls/lib/curve_advisory_helper.py`:

```python
# was:
try:
  self.enabled = self._params.get_bool("CurveSpeedAdvisory")
except UnknownKeyName:
  self.enabled = True

# becomes:
self.enabled = not os.path.exists("/data/curve_advisory_disabled")
```

Same change, same file, at both call sites (`__init__` and `_read_params`) — and the
`_read_params` polling loop stays, so toggling the file takes effect live within one
`PARAMS_UPDATE_PERIOD` tick, no reboot needed, matching how Phase 3 arming already
behaves.

`sunnypilot/selfdrive/controls/lib/lead_closing_advisory_helper.py`: identical pattern,
`/data/lead_closing_advisory_disabled`.

`sunnypilot/selfdrive/controls/lib/lead_closing_test_guidance_helper.py`: opt-**in** this
time, preserving the current default-off:

```python
self.enabled = os.path.exists("/data/lead_closing_test_guidance_enabled")
```

Three files, ~6 lines changed total, zero new imports beyond `os` (already imported
everywhere in this codebase), zero UI surface, zero compiled-binary dependency.

### Why this is low-risk relative to the incident that made this project cautious

- Doesn't touch `selfdrive/ui/` — different subsystem entirely from what caused Q13.
- Doesn't add new continuous file I/O inside a control loop — it's a read, gated behind
  the existing `PARAMS_UPDATE_PERIOD` throttle already in both files today, same pattern
  already proven safe for Phase 3 arming (which polls its own flag files continuously,
  same order of magnitude, no incident).
- Doesn't change default behavior for the car this already runs on — same on/off state
  before and after, for anyone who never touches the new files.

## Explicitly out of scope for this plan — not forgotten, just ruled out for now

- **The OSM map-region silent-failure trap.** A real, on-screen "no map data for your
  location" warning would be the right fix — but that's a UI change, ruled out. No
  UI-free substitute is proposed here; flagging this as a separate, still-open problem
  rather than pretending this plan solves it.
- **A first-boot disclaimer.** Also inherently a UI/onboarding-screen change. Not
  addressed here.
- **Making actuation reachable without SSH.** Deliberately left alone — per the review in
  `progress.md`, this may be the *correct* design given the safety stakes, not a gap to
  close reflexively. A separate, explicit decision, not bundled into this fix.

## Staged rollout, matching this project's own standing discipline

1. Implement the three-file change on a throwaway branch first, not directly on
   `phase3-actuation`.
2. Bench-verify: SSH in, confirm `curve_advisory_helper.py`'s `self.enabled` flips
   correctly when the flag file is created/removed, parked, before any driving.
3. Merge onto `phase3-actuation` only after that holds — this *is* the daily driver, so
   land the change there once it's confirmed safe, the same way every other Phase 3 change
   in this project has been verified before trusting it live.
4. One short, low-stakes drive confirming the advisories still fire exactly as they do
   today (unchanged behavior) before calling it done.
5. Only then does `INSTALL.md`'s "how to turn off the advisories" section go from
   aspirational to real.

## Self-test checklist — for the two of you to run as "laymen," once implemented

Not for a stranger — this is the internal dry run mentioned in conversation: follow
`INSTALL.md` (see the companion file at the repo root) as if neither of you has touched
this project before, and note anywhere the instructions are wrong, unclear, or assume
knowledge a real stranger wouldn't have.

- [ ] Follow `INSTALL.md`'s install-link step exactly as written. Does the URL work
      first try? (Flag if the GitHub account rename changed the working URL pattern —
      confirm the actual live install path once the rename is done, don't assume the old
      `install.sunnypilot.ai/fork/<old-username>/<branch>` pattern still resolves.)
- [ ] After first boot, with zero manual steps taken: do the curve and lead advisories
      fire on a normal drive, matching what's described as "on by default"?
- [ ] SSH in using only the instructions in `INSTALL.md` (not prior knowledge) —
      `touch /data/curve_advisory_disabled`. Confirm the advisory stops firing within
      one drive cycle, no reboot.
- [ ] Remove the file, confirm the advisory comes back.
- [ ] Repeat for `/data/lead_closing_advisory_disabled`.
- [ ] Confirm `/data/lead_closing_test_guidance_enabled` actually turns that guidance on
      (the thing that was previously broken) — this is the one regression test that
      directly proves the fix, not just "nothing broke."
- [ ] Confirm actuation stays fully inert with no flag files present — this should be
      unchanged, but re-verify rather than assume.
- [ ] Note every point where `INSTALL.md` assumes something a genuine first-timer
      wouldn't know (comma device setup, harness install, what "SSH in" even means
      concretely) — flag for a follow-up pass, don't fix silently.
