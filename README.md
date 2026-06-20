# Garden Watering Scheduler — mk2 (valve-timer offload)

A Home Assistant dashboard card that schedules sequential garden watering across
**1–5 configurable valve zones**, with **two independent schedules** out of the box.

mk2 is a re-architecture of the original
[Garden Watering Scheduler](https://github.com/rhamblen/garden-watering-scheduler):
instead of holding each zone open with its own `delay:`, mk2 **offloads the close to
the valve's own countdown timer** and waits for the valve to report `off`. This makes
runs **survive HA restarts**, **inherit the valve's volume safety cutoff for free**,
and **self-correct the countdown** when a valve is closed early.

---

## ⚠️ Requirement: this version only works with the Zigbee valve tiles

> **mk2 requires the [Zigbee Smart Water Valve card](https://github.com/rhamblen/Zigbee-smart-water-valve) (v0.3.0+) on every valve you schedule.**
> It delegates each zone's close to that card's per-valve countdown timer
> (`timer.<prefix>_timer` + the `timer.finished` → `switch.turn_off` automation —
> the [integration contract](docs/integration-contract.md)).
>
> **If your valves are plain on/off switches with no timer, use
> [mk1](https://github.com/rhamblen/garden-watering-scheduler) instead.** mk1
> self-manages the watering delay and works with **any** valve that has on/off
> capability — it has no dependency on the valve tile.

| | **mk1** (`garden-watering-scheduler`) | **mk2** (this repo) |
|---|---|---|
| Valve requirement | **Any** on/off switch | **Zigbee Smart Water Valve tile** (timer + finished automation) |
| Who closes the valve | The scheduler's own `delay:` then `turn_off` | The valve card's `timer.finished` automation |
| Survives HA restart mid-run | ❌ delay is lost | ✅ timer is `restore: true` |
| Volume safety cutoff | ✗ not available | ✅ inherited (waits on `valve → off`) |
| Countdown if a valve is shut early | stays wrong | self-corrects each zone |

---

## How a run works (mk2)

Per zone, in slot order 1 → 5 (skipping blank/0 slots):

1. Open the valve.
2. `timer.start` the valve's `timer.<prefix>_timer` for the zone duration.
3. Wait for the valve to report `off` — by the valve card's `timer.finished`
   automation, a volume cutoff, a manual stop, or a fault — then move to the next zone.
4. Backstop: if the valve is still open after `duration + 30 s`, mk2 force-closes it.

There is **no fixed settle delay** between zones (mk1 needed one): waiting on the
actual `→ off` state is the confirmation that the just-closed valve has propagated, so
the single-valve cap guard for the next zone is always correct.

The card shows a live **Time remaining** countdown that counts down to a re-stamped
`run_end` timestamp, so closing a valve early (manual override) immediately corrects
the displayed total.

## Features

- **Two independent schedules (A & B)** — each with its own days, start time, and
  1–5 valve zones (`garden_a_*` / `garden_b_*`).
- **Single-valve cap (FIFO)** — only one garden valve open system-wide; a zone that
  would overlap is skipped, not queued (keeps line pressure up).
- **Automatic rain cancel** — 12 h actual-rain lookback + 24 h forecast probability,
  shared across both schedules; with a persistent notification and manual 🌧 override.
- **Winter shutdown** — ❄ suspends every schedule at once.
- **▶ Start now / ■ Stop** header controls.

## Install

See [INSTALLATION.md](INSTALLATION.md). In short: set up the valve tiles + their
timers first, create the helpers ([helpers.md](helpers.md)), install
`scripts.yaml`, `schedule-automations.yaml`, `rain-automations.yaml`, and add
`card-a.yaml` / `card-b.yaml` to a dashboard.

## Requirements

- Home Assistant with `custom:html-template-card` (HACS).
- The [Zigbee Smart Water Valve card](https://github.com/rhamblen/Zigbee-smart-water-valve)
  v0.3.0+ deployed for every scheduled valve (timer + finished automation).
- A `weather.*` entity with hourly forecasts (for rain cancel).

## Companion projects

- [Zigbee Smart Water Valve card](https://github.com/rhamblen/Zigbee-smart-water-valve) — the valve tile mk2 offloads to (defines the valve entities and the timer contract).
- [Garden Watering Scheduler (mk1)](https://github.com/rhamblen/garden-watering-scheduler) — the self-managed-delay version for any on/off valve.
