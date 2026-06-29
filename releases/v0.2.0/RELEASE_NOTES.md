# v0.2.0 — Backstop retry loop + emergency water meter shutoff

## What changed

### Backstop retry loop
Previously, if a valve failed to close after its scheduled duration, mk2 made a single
force-close attempt, waited 15 seconds, and moved on regardless.

In v0.2.0, the backstop retries every 15 seconds for up to **30 minutes (120 attempts)**
before giving up. Each iteration sends `turn_off` + `timer.cancel` and re-checks the
valve state.

### Emergency water meter shutoff
If the valve is still reporting `on` after all 120 retries, mk2 turns off
`switch.garden_water_meter` to stop flow at the source.

Applies to all three schedules — A, B, and C.

## Files changed

- `scripts.yaml` — backstop block updated in all 5 slots of both schedule A and B
  (template version; live HA instance covers schedules A, B, and C)

## Upgrade

Replace `scripts.yaml` in your HA instance with this version, or apply via Claude:

> Update my garden watering scheduler scripts to v0.2.0 from
> https://github.com/rhamblen/garden-watering-scheduler-mk2
