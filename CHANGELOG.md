# Changelog

All notable changes to this project are documented here.
The format is based on [Keep a Changelog](https://keepachangelog.com/).

## [0.2.0] — 2026-06-29

### Changed

- **Backstop retry loop** — if a valve fails to close, mk2 now retries a force-close every
  15 seconds for up to **30 minutes (120 attempts)** instead of making a single attempt and
  moving on.
- **Emergency water meter shutoff** — after 120 failed retries with the valve still reporting
  `on`, `switch.garden_water_meter` is turned off as a last resort to stop flow at the
  source. Applies to all three schedules (A, B, C).

---

## [0.1.0] — 2026-06-20

First release of the **mk2** scheduler — a re-architecture of
[garden-watering-scheduler](https://github.com/rhamblen/garden-watering-scheduler)
that offloads each zone's close to the Zigbee Smart Water Valve card's own countdown
timer.

### Added
- **Valve-timer offload engine** (`scripts.yaml`): each zone opens the valve, starts
  `timer.<prefix>_timer`, and waits for the valve to report `off` (closed by the valve
  card's `timer.finished` automation, a volume cutoff, a manual stop, or a fault).
  A timeout backstop force-closes the valve if the timer contract is missing.
- **Self-correcting countdown**: `garden_a/b_run_end` is re-stamped at every zone, so
  the card countdown drops to the true remaining time when a valve is shut early.
- **Two independent schedules (A & B)** from the first release — namespaced
  `garden_a_*` / `garden_b_*` (no singleton migration stage).
- Single-valve FIFO cap, automatic rain cancel (shared), winter shutdown (shared),
  ▶ Start now / ■ Stop — carried over from mk1.
- `docs/integration-contract.md` documenting the valve timer + finished automation
  dependency.

### Changed (vs mk1)
- Zone control switched from self-managed `delay:` to timer offload + `wait_template`
  on the valve closing.
- **Removed the 10 s inter-zone settle delay** — waiting on the actual `→ off` state
  makes it unnecessary (and removes the `settle_s` term from the card countdown).
- Helper `garden_X_run_started` → `garden_X_run_end`.

### Requirements
- **Requires the [Zigbee Smart Water Valve card](https://github.com/rhamblen/Zigbee-smart-water-valve)
  v0.3.0+ on every scheduled valve.** For plain on/off valves, use mk1.
