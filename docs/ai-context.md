# AI Context — Garden Watering Scheduler mk2

Reference for Claude/AI assistants navigating this repo. mk2 is a re-architecture of
[garden-watering-scheduler (mk1)](https://github.com/rhamblen/garden-watering-scheduler).
Read this before editing.

## What mk2 is

Same UX as mk1 (1–5 zone sequential watering, two independent schedules A/B, rain
cancel, winter shutdown, start-now/stop, live countdown) **but the watering engine is
different**: mk2 **offloads each zone's close to the Zigbee Smart Water Valve card's
own countdown timer** instead of holding the valve open with its own `delay:`.

**Hard requirement:** every scheduled valve must have the valve card's
`timer.<prefix>_timer` + a `timer.finished` → `switch.turn_off` automation (valve card
v0.3.0+). See `docs/integration-contract.md`. mk1 remains the choice for plain on/off
valves.

## File structure

```
/
├── README.md                  — overview; leads with the Zigbee-tile requirement + mk1/mk2 table
├── CHANGELOG.md               — [0.1.0]
├── INSTALLATION.md            — Claude-assisted (Option A) then manual (Option B); coexistence; troubleshooting
├── hacs.json                  — points at card-a.yaml
├── .gitignore
├── docs/
│   ├── ai-context.md          — this file
│   └── integration-contract.md— the valve timer/finished contract mk2 depends on
├── card-a.yaml / card-b.yaml  — dashboard cards (garden_a_* / garden_b_*)
├── scripts.yaml               — script.garden_a/b_watering_sequence — THE OFFLOAD ENGINE
├── schedule-automations.yaml  — automation.garden_a/b_watering_schedule
├── rain-automations.yaml      — 3 shared rain automations (weather entity parameterised)
├── helpers.md                 — 25 per-schedule helpers ×2 + 4 shared
└── releases/v0.1.0/           — snapshot + RELEASE_NOTES.md
```

## The offload engine (`scripts.yaml`) — the heart of mk2

Per zone (slot 1–5, in order, skipping duration 0 / cap-blocked):

1. `if` guard: `numeric_state` duration > 0 **and** the FIFO single-valve cap template
   (no garden valve `on` across both `['a','b']` namespaces' `*_valve_N_entity`).
2. `variables`: `sw` (the switch entity from `input_text.garden_X_valve_N_entity`),
   `tmr` = `"timer." ~ sw.split('.')[1] ~ "_timer"`, `secs` = duration×60.
3. Re-stamp `input_datetime.garden_X_run_end` = `now() + sum(durations of slots N..5)`
   → self-correcting countdown.
4. `homeassistant.turn_on` `sw`.
5. `timer.start` `tmr` with `duration` formatted `HH:MM:SS` (cv.time_period needs the
   string form, not bare seconds).
6. `wait_template "{{ is_state(sw,'on') }}"` (timeout 30 s, continue) — handles Zigbee
   report lag so step 7 doesn't pass instantly on a stale `off`.
7. `wait_template "{{ is_state(sw,'off') }}"` (timeout `secs+30`, continue) — advances
   on whatever closes the valve.
8. Backstop `if is_state(sw,'on')`: force `turn_off` + `timer.cancel` +
   `wait_template off` (so the next zone's cap guard is correct).

Step 0 (before the slots) stamps `run_end` with the full remaining run so the countdown
is right from t=0. **Mode: single.** The cap guard hardcodes `['a','b']` — extend it in
**both** scripts to add a schedule C/D.

### Why no settle delay
mk1 needed `delay: {seconds:10}` after each `turn_off` because it moved on before HA
observed the Zigbee valve report `off` (next cap guard saw it still `on` → skipped the
zone; the mk1 v0.6.1 bug). mk2 waits on the real `→ off` state, so the off-state is
already registered. The wait *is* the settle.

## Cards (`card-a.yaml` / `card-b.yaml`)

Identical to mk1's v0.6.2 cards **except** the countdown source:
- mk1: `run_ts = state_attr('…run_started','timestamp')`; `end_ts = run_ts + total_d*60 + settle_s`.
- mk2: `end_ts = state_attr('…garden_X_run_end','timestamp')|int(0)`; `rem_s = end_ts - now`.
  No `settle_s`. The client-side ticker (`window.gwsT_a` / `_b`) re-syncs to `end_ts` on
  every re-render, so re-stamping `run_end` mid-run makes the displayed time jump to the
  corrected value. CSS prefix `.gws-`. Watch the `{{%` defensive-templating rule (a space
  between JS/CSS `{` and a following `{%`).

## Carried over unchanged from mk1
`schedule-automations.yaml` (with the shared `garden_winter_shutdown == off` condition),
`rain-automations.yaml` (recorder + OR-across-schedules auto-check + latest-start reset;
weather entity parameterised, `# << SET WEATHER ENTITY`), the FIFO cap guard, and the
shared helpers (`garden_rain_cancel`, `garden_rain_threshold`, `garden_last_rain`,
`garden_winter_shutdown`).

## Helpers
50 per-schedule (25 × A/B) + 4 shared. The only difference from mk1 is
`garden_X_run_started` → `garden_X_run_end`. No `timer.*` helpers here — timers belong
to the valve card and are derived by convention.

## How to publish a new version
1. Update the relevant root files. 2. Snapshot into `releases/vX.Y.Z/` with a
`RELEASE_NOTES.md`. 3. Update `CHANGELOG.md`. 4. Commit, tag `vX.Y.Z`, push to master.
5. `gh release create vX.Y.Z -F releases/vX.Y.Z/RELEASE_NOTES.md` (gh at
`C:\Program Files\GitHub CLI\gh.exe` if not on PATH). Always publish the GitHub Release
after push + tag.
