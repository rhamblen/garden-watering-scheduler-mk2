# Integration contract — mk2 ⇄ Zigbee Smart Water Valve card

mk2 is a thin **orchestrator**. It does not time or close valves itself — it hands
each zone's close to the valve's own countdown timer and waits for the valve to
report `off`. That delegation depends on a small, stable contract the
[Zigbee Smart Water Valve card](https://github.com/rhamblen/Zigbee-smart-water-valve)
provides from **v0.3.0** onward.

## What the valve side must provide (per scheduled valve)

For a valve switch `switch.<prefix>` (e.g. `switch.tap_lhs_lower_lawn_green`):

| Object | Entity / definition | Source |
|--------|--------------------|--------|
| Countdown timer | `timer.<prefix>_timer` (e.g. `timer.tap_lhs_lower_lawn_green_timer`), `restore: true` | Valve card v0.3.0 |
| Auto-close on finish | Automation: trigger `timer.finished` for that timer → `switch.turn_off` on `switch.<prefix>` | Valve card v0.3.0 |

The **naming convention is the contract**: `switch.<prefix>` ⇒ `timer.<prefix>_timer`.
mk2 derives the timer entity from the configured switch entity by
`"timer." ~ switch.split('.')[1] ~ "_timer"`. No timer helper is created on the
scheduler side.

## What mk2 does with it (per zone)

1. `homeassistant.turn_on` the valve switch.
2. `timer.start` on `timer.<prefix>_timer` for the zone's duration.
3. Wait for the valve to report `on` (handles Zigbee report lag), then wait for it to
   report `off`. **Whatever** closes the valve advances the sequence:
   - the valve card's `timer.finished` automation (normal case),
   - the valve card's **volume safety cutoff** (v1.0.0 pool fill) — inherited for free,
   - a manual ■ Stop or a fault.
4. **Backstop:** if the valve is somehow still `on` after `duration + 30 s`, mk2
   force-closes it (`switch.turn_off` + `timer.cancel`). This keeps mk2 safe even if a
   valve is mis-configured and the contract is not actually present.

## Properties this buys

- **Restart-survivable closes** — the `timer` is `restore: true`, so a valve closes
  itself even if HA restarts mid-zone (a self-managed `delay:` cannot).
- **Volume safety inherited** — because mk2 waits on `valve → off`, the valve's volume
  cutoff doubles as a backstop with zero coupling to the pool helpers.
- **No double-ownership of the close** — the valve card's `timer.finished` automation
  is the single normal closer; mk2 only force-closes in the timeout fallback.

## Versioning

This contract is documented on the valve card side from **v1.1.0** (docs-only; the
timer + finished automation themselves are unchanged since v0.3.0). Breaking the
naming convention or removing the finished automation would break mk2 — treat both as
a public interface.

## If you don't have it

Plain on/off valves with no timer should use **mk1**
(`garden-watering-scheduler`), which self-manages the delay and works with any switch.
