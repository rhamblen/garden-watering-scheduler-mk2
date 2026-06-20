# mk2 helpers (A & B)

mk2 ships **two independent schedules from the first release** (no singleton stage),
so create both the `garden_a_*` and `garden_b_*` sets up front. Per the Option 3
hybrid-namespace design, only **per-schedule** state is namespaced; house-wide
concepts stay **shared** (single instance).

> ⚠️ **Prerequisite — Zigbee valve tiles.** mk2 hands the close of each zone to the
> valve's own countdown timer, so **every valve you schedule must have the
> [Zigbee Smart Water Valve card](https://github.com/rhamblen/Zigbee-smart-water-valve)
> timer set up** (a `timer.<prefix>_timer` helper + the `timer.finished` →
> `switch.turn_off` automation, both from valve card v0.3.0+). See
> `docs/integration-contract.md`. If you have plain on/off valves with no timer,
> use **mk1** (`garden-watering-scheduler`) instead — it self-manages the delay and
> works with any switch.

---

## Per-schedule helpers — 25 per schedule

Create the full set for each of `a` and `b` (the table shows `a`; repeat with `b`).

| Helper | Type | Notes |
|--------|------|-------|
| `input_boolean.garden_a_water_mon` … `_sun` | Toggle × 7 | Day-of-week schedule |
| `input_select.garden_a_water_start_time` | Dropdown | 48 options `00:00`…`23:30` |
| `input_text.garden_a_valve_1_entity` … `_5_entity` | Text × 5 | Valve switch entity id per slot (blank = slot unused) |
| `input_text.garden_a_valve_1_name` … `_5_name` | Text × 5 | Zone display name per slot |
| `input_number.garden_a_valve_1_duration` … `_5_duration` | Number × 5 | 0–60, step 5 (0 = excluded) |
| `input_boolean.garden_a_schedule_armed` | Toggle | Schedule A active/inactive |
| `input_datetime.garden_a_run_end` | Date+time | **Re-stamped at every zone** to the projected run end; powers A's self-correcting countdown |

**Total per-schedule helpers: 25 × 2 = 50.**

> The only helper that differs from mk1 is `garden_X_run_end` (mk1 used
> `garden_X_run_started`). mk2 stamps the *projected end* and re-stamps it each zone,
> so the card countdown self-corrects when a valve closes early.

---

## Shared helpers — unchanged, single instance

Do **not** namespace these. One rain skip / one winterise applies to every schedule.

| Helper | Type | Why shared |
|--------|------|-----------|
| `input_boolean.garden_rain_cancel` | Toggle | Rain is a property of the weather, not a schedule |
| `input_number.garden_rain_threshold` | Number (0–100, step 5) | One forecast threshold |
| `input_datetime.garden_last_rain` | Date+time | One 12 h actual-rain lookback (seed to an old value, e.g. 2020-01-01) |
| `input_boolean.garden_winter_shutdown` | Toggle | ❄ suspends **everything at once** |

The weather entity (e.g. `weather.met_office_fleet`) is also shared and set in
`rain-automations.yaml`.

---

## No per-valve timer helpers here

mk2 does **not** create any `timer.*` helpers — those belong to the valve card and
are derived by convention from the valve switch entity
(`switch.<prefix>` → `timer.<prefix>_timer`). Set them up on the valve side per the
valve card's INSTALLATION (its Phase 3 / v0.3.0).

---

## Objects to install alongside the helpers

| File | Creates |
|------|---------|
| `scripts.yaml` | `script.garden_a_watering_sequence`, `script.garden_b_watering_sequence` (the offload engine + single-valve cap guard) |
| `schedule-automations.yaml` | `automation.garden_a_watering_schedule`, `automation.garden_b_watering_schedule` (with the shared winter-off condition) |
| `rain-automations.yaml` | The 3 shared rain automations (recorder + auto-check + daily reset, generalised to both schedules) |
| `card-a.yaml`, `card-b.yaml` | The two dashboard cards |

---

## Coexisting with / migrating from mk1

mk2's canonical entity names are `garden_a_*` / `garden_b_*` — the **same names mk1
uses**. To run mk2 **alongside** a live mk1 (recommended while you prove mk2 out),
deploy mk2 under a distinct namespace by find-replacing `garden_a_` → `garden2_a_`
and `garden_b_` → `garden2_b_` across all mk2 files (leave the shared `garden_rain_*`,
`garden_last_rain`, `garden_winter_shutdown` names untouched so both systems share one
rain/winter state), and the JS globals `window.gwsT_a/_b` → `window.gwsT2_a/_b`.
**Arm only one system at a time** for the same physical valves during transition — the
single-valve cap is enforced within a system, not across the two. Once mk2 is trusted,
retire the mk1 scripts/automations/cards and (optionally) rename `garden2_*` back to
`garden_*`.
