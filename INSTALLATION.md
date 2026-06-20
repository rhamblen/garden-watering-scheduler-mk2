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
> Then create the 50 per-schedule helpers + 4 shared helpers, install
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
