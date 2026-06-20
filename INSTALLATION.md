# Installation — Garden Watering Scheduler mk2

> **Before you start — mk2 requires the Zigbee valve tiles.** Each valve you schedule
> must already have the [Zigbee Smart Water Valve card](https://github.com/rhamblen/Zigbee-smart-water-valve)
> v0.3.0+ set up, including its per-valve **timer** (`timer.<prefix>_timer`) and the
> **`timer.finished` → `switch.turn_off`** automation. mk2 hands the close of each zone
> to that timer. If your valves are plain on/off switches, install
> [mk1](https://github.com/rhamblen/garden-watering-scheduler) instead.

---

## Option A — Claude-assisted (recommended)

If you use Claude with Home Assistant access, paste this:

> Install the Garden Watering Scheduler **mk2** for me. Reference
> https://github.com/rhamblen/garden-watering-scheduler-mk2 for context. First confirm
> each valve I want to schedule has its `timer.<prefix>_timer` helper and a
> `timer.finished` auto-close automation (the Zigbee Smart Water Valve card contract).
> Then create the 50 per-schedule helpers (25 each for A and B) + 4 shared helpers, install
> `scripts.yaml`, `schedule-automations.yaml`, and `rain-automations.yaml`, discover my
> `weather.*` forecast entity and wire it into the rain automations, and add
> `card-a.yaml` and `card-b.yaml` to my dashboard.

Claude should:
1. **Verify the contract** — for each valve, check `timer.<prefix>_timer` exists and a
   `timer.finished` automation closes `switch.<prefix>`. Flag any valve missing it.
2. Create the helpers from [helpers.md](helpers.md).
3. Install the scripts + automations.
4. **Discover the weather entity** (don't hardcode) — find a `weather.*` entity with
   hourly forecast support and set it in the two `# << SET WEATHER ENTITY` spots in
   `rain-automations.yaml`.
5. Add the two cards to a dashboard.

---

## Option B — Manual

### 1. Valve prerequisite (per scheduled valve)
Confirm on the valve side: `timer.<prefix>_timer` exists, and an automation triggered
by `timer.finished` for that timer calls `switch.turn_off` on `switch.<prefix>`. These
come from the Zigbee Smart Water Valve card v0.3.0. See
[docs/integration-contract.md](docs/integration-contract.md).

### 2. Helpers
Create the helpers in [helpers.md](helpers.md): 25 per schedule × 2 (`garden_a_*`,
`garden_b_*`) + the 4 shared house-wide helpers. Seed `input_datetime.garden_last_rain`
to an old value (e.g. `2020-01-01 00:00`).

### 3. Scripts & automations
- `scripts.yaml` → create `script.garden_a_watering_sequence` and
  `script.garden_b_watering_sequence`.
- `schedule-automations.yaml` → the two schedule automations.
- `rain-automations.yaml` → the 3 shared rain automations. **Set your weather entity**
  in the two `# << SET WEATHER ENTITY` spots (must support hourly forecasts). To find
  it: Developer Tools → States → search `weather.` and pick your forecast provider.

### 4. Cards
Add `card-a.yaml` and `card-b.yaml` to a dashboard (Edit dashboard → Add card → Manual).
Requires `custom:html-template-card` (HACS).

### 5. Configure
On each card: pick days, set the start time, fill the valve slots (entity + name +
duration), and press **▶ Go** to arm.

---

## Running alongside mk1 (coexistence)

mk2 uses the same `garden_a_*` / `garden_b_*` names as mk1. To run both at once while
testing, deploy mk2 under a `garden2_a_*` / `garden2_b_*` namespace (and
`window.gwsT2_a/_b`) — see the "Coexisting with / migrating from mk1" section in
[helpers.md](helpers.md). **Arm only one system at a time** for the same physical
valves during the transition.

---

## Adding another schedule (the second tile, and beyond)

> ### ⚠️ Do not duplicate the card
> A second copy of the **same** card is **not** a second schedule. Every copy reads and writes the
> same `garden_a_*` helpers, so two copies control **one** schedule — change the days on one and they
> change on the other, and their client-side countdowns fight over the one `window.gwsT_a` global. An
> independent schedule needs its **own namespaced helpers**, its **own card** (with its own
> `window.gwsT_*`), and — for a third schedule onward — its **own** script + automation.

mk2 ships **two** schedules: `card-a.yaml` (`garden_a_*`) and `card-b.yaml` (`garden_b_*`), and
`scripts.yaml` / `schedule-automations.yaml` already define **both** A and B. Per-schedule state
(days, start time, valve slots 1–5, armed, `run_end`) is namespaced; the rain helpers/automations and
`garden_winter_shutdown` stay **shared** house-wide (winterise once → all schedules suspended; one
rain skip → all schedules skip). Overlapping runs follow the FIFO single-valve cap.

### Schedule B — the second tile

If you added `card-b.yaml` in the base install, B is already live — just configure it on the **(B)**
card. To add B to an A-only install:

- **Ask Claude:** *"Add the second schedule (B) to my Garden Watering Scheduler mk2 — create the 25
  `garden_b_*` helpers and add `card-b.yaml` next to my A card. `scripts.yaml` and
  `schedule-automations.yaml` already include B. Keep rain cancel and winterise shared."*
- **Manual:**
  1. Create the 25 `garden_b_*` helpers ([helpers.md](helpers.md)). Populate each
     `garden_b_valve_N_entity` (+ `_name`) for every zone you want to appear — a blank entity renders
     no zone row; durations/days can start at 0/off until you set them on the card.
  2. Add a second Manual card with `card-b.yaml`. It already uses `garden_b_*` and `window.gwsT_b`, so
     **never hand-copy A's content for B** or the two countdowns will fight over one timer global.

  `scripts.yaml` and `schedule-automations.yaml` already contain the B objects, so nothing else to do.

### A third schedule (C) and beyond

Repeat for a `garden_c_*` namespace: create 25 `garden_c_*` helpers, clone a sequence script and a
schedule automation into the `c` namespace, and add a `card-c.yaml` (find-replace `garden_a_` →
`garden_c_`, `window.gwsT_a` → `window.gwsT_c`, and the title). The **only** cross-schedule edits:
extend the cap-guard namespace list `['a','b']` → `['a','b','c']` in **every** sequence script, and
add `'c'` to the two templated rain automations (auto-cancel check + daily reset). For ~4+ schedules,
graduate to a parameterised "schedule slot" model instead of named clones.

---

## Troubleshooting

- **A zone is skipped** — the single-valve cap saw another garden valve already `on`
  (manual, pool fill, or the other schedule). Only one garden valve runs at a time.
- **A valve opens but never closes** — its `timer.<prefix>_timer` / `timer.finished`
  automation is missing or misnamed. mk2's backstop will force-close it after
  `duration + 30 s`, but fix the valve-side contract so the normal close works.
- **Pool fill + schedule on the same valve** — mk2 is time-based and never sets
  `pool_fill_active`, so it can't trip the valve card's v1.0.1 auto-reopen. But if you
  leave **pool mode on manually** on a valve a schedule then runs, the scheduled close
  can be undone by auto-reopen. **Don't schedule a valve that is mid-pool-fill.**
- **Countdown looks off after a manual early close** — it self-corrects at the next
  zone boundary (when `run_end` is re-stamped).
- **Two cards change together / their countdowns fight** — you copied a card instead of using a
  namespaced one. Each schedule needs its own `garden_X_*` helpers and its own card; use `card-b.yaml`
  (it uses `garden_b_*` + `window.gwsT_b`), never a hand-copy of card A. See *Adding another schedule*.
