# v0.1.0 — Garden Watering Scheduler mk2 (valve-timer offload)

First release of **mk2**, a re-architecture of
[garden-watering-scheduler](https://github.com/rhamblen/garden-watering-scheduler).
Instead of holding each zone open with its own `delay:`, mk2 **offloads the close to
the Zigbee Smart Water Valve card's own countdown timer** and waits for the valve to
report `off`.

## ⚠️ Requirement
**mk2 only works with the [Zigbee Smart Water Valve tiles](https://github.com/rhamblen/Zigbee-smart-water-valve)
(v0.3.0+).** Each scheduled valve needs its `timer.<prefix>_timer` helper and the
`timer.finished` → `switch.turn_off` automation. **For plain on/off valves, use mk1** —
it self-manages the delay and works with any switch.

## What's new vs mk1
- **Offload engine** (`scripts.yaml`): per zone — open valve → `timer.start` →
  wait for the valve to report `off` (closed by the valve card's `timer.finished`
  automation, a volume cutoff, a manual stop, or a fault) → next zone. A timeout
  backstop force-closes the valve if the timer contract is missing.
- **Restart-survivable runs** — the valve closes itself via its `restore: true` timer
  even if HA restarts mid-zone.
- **Volume safety inherited for free** — because it waits on `valve → off`, a volume
  cutoff also advances the sequence (no coupling to the pool helpers).
- **Self-correcting countdown** — `garden_a/b_run_end` is re-stamped each zone, so a
  valve shut early immediately corrects the displayed remaining time.
- **No 10 s inter-zone settle delay** — waiting on the real `→ off` state replaces it.
- **Two independent schedules (A & B)** from the first release (`garden_a_*` /
  `garden_b_*`).
- Carried over from mk1: single-valve FIFO cap, shared automatic rain cancel, winter
  shutdown, ▶ Start now / ■ Stop.

## Install
See [INSTALLATION.md](https://github.com/rhamblen/garden-watering-scheduler-mk2/blob/master/INSTALLATION.md).
Set up the valve tiles + timers first, create the helpers
([helpers.md](https://github.com/rhamblen/garden-watering-scheduler-mk2/blob/master/helpers.md)),
install `scripts.yaml` / `schedule-automations.yaml` / `rain-automations.yaml`, and add
`card-a.yaml` / `card-b.yaml` to a dashboard.

## Helper change
`garden_X_run_started` (mk1) → `garden_X_run_end` (mk2). No `timer.*` helpers are
created on the scheduler side — they belong to the valve card and are derived by
convention (`switch.<prefix>` → `timer.<prefix>_timer`).
