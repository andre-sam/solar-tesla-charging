# Solar-Aware Tesla Charging (Home Assistant Blueprint)

Ramp-first solar charging control for a Tesla Wall Connector using any
grid-power sensor pair (Enphase, Shelly EM, Powerwall, etc.). One
blueprint, one automation, every behaviour:

- Ramps current between 1 A and charger max. Never stop/starts unnecessarily.
- Holds at minimum through short cloud gaps before ending a session,
  with a configurable grace period; if export recovers, the grace
  timer aborts automatically.
- Instant grid-import guard: trims current the moment any appliance
  causes net import.
- Fast recovery after a brief import spike. The guard stamps an
  `input_datetime` the same automation reads to skip its stability
  delay so a kettle pulse doesn't pin charging low.
- Uses Tesla Fleet API's 1 A floor (about 690 W on 3-phase), not the
  app's 5 A.
- Works with 1-, 2-, or 3-phase installs (configurable line voltage
  and phase count).
- SOC-aware: stops at the lower of the configured cap and the Tesla
  car-side charge limit.
- Lifecycle notifications: plug-in (whether solar charging is armed
  or not), enable while plugged in, disable mid-session.
- Session-end notifications for SOC cap, optional hard window-end
  stop, and solar-exhaustion stop.
- Wakes an ambiguous-status Tesla on resume so a mid-session sleep
  doesn't block restart.
- Optional **Solcast forecast boost**: raises the SOC cap on a sunny
  day when the next 1 to 6 days are forecast to be poor, so you
  stash extra charge before bad weather. Disabled by default;
  activates only when Solcast sensors are configured.

## Blueprints

| Blueprint | Purpose |
|---|---|
| `solar_tesla_controller.yaml` | Single all-in-one automation. Controls ramp / start / stop, runs the import-spike trim, the below-minimum grace stop, and the plug/toggle lifecycle notifications. |
| `solar_tesla_stop_at_soc.yaml` | **Deprecated.** The controller now handles the SOC cap stop itself. Kept only for users who already imported it. |

Earlier versions of this repo split the behaviour across four
blueprints (Controller + Grace Stop + Import Guard + Plugged-In
Notify). The consolidated controller absorbs all four. If you're
upgrading, delete the three retired blueprints and the automations
they powered, then import the new controller and configure it once.

## Import

In Home Assistant: **Settings -> Automations -> Blueprints -> Import
Blueprint**, then paste the raw GitHub URL of
`solar_tesla_controller.yaml`.

## Required entities

You need these from your installation (any integration, any names):

- A **grid export power** sensor in W (at or below 0 when exporting:
  signed net or dedicated export counter)
- A **grid import power** sensor in W (at or above 0 when importing:
  dedicated import counter, or the same signed sensor as above)
- A **wall connector power** sensor in W or kW
- A **wall connector vehicle-connected** binary sensor
- A **wall connector contactor-closed** binary sensor
- A **wall connector status** sensor (text)
- A **Tesla charge current** `number` entity (amps)
- A **Tesla charge switch** (on/off)
- A **Tesla battery SOC** sensor (%)
- An **`input_boolean`** you create yourself as the master enable
  toggle (e.g. `input_boolean.solar_charging_enabled`)
- A second **`input_boolean`** for the below-minimum tracker
  (e.g. `input_boolean.solar_charging_below_min`). The controller
  toggles this internally; you do not need to interact with it.
- Two **`input_datetime`** helpers (time only) for the charging
  window (e.g. `input_datetime.solar_window_start`,
  `input_datetime.solar_window_end`). Expose these on your dashboard
  to change the window without editing the automation.

Optional:

- A **Tesla charge limit** `number` entity (controller falls back to
  the configured SOC cap if absent)
- A **wake-up** `button` for the Tesla integration
- An **`input_datetime`** (date + time) used to stamp the last
  import-spike trim, enabling fast recovery after brief import
  spikes (e.g. `input_datetime.solar_charging_last_guard_trim`)
- A **notify service** (e.g. `notify.mobile_app_phone`) for session
  and lifecycle notifications
- One or more **`input_boolean`** entities that gate other power-
  hungry automations (AC, pool pump, etc.) you want the controller
  to pause while the Tesla SOC is low. See [Load prioritisation](#load-prioritisation-optional).
- **Solcast PV Forecast** daily-total sensors for the forecast boost.
  Pick today, tomorrow, and as many of `day_3` to `day_7` as you
  want (the lookahead input chooses how far ahead to inspect):
  `sensor.solcast_pv_forecast_forecast_today`,
  `sensor.solcast_pv_forecast_forecast_tomorrow`,
  `sensor.solcast_pv_forecast_forecast_day_3`,
  `sensor.solcast_pv_forecast_forecast_day_4`,
  `sensor.solcast_pv_forecast_forecast_day_5`,
  `sensor.solcast_pv_forecast_forecast_day_6`,
  `sensor.solcast_pv_forecast_forecast_day_7`

## SOC caps are local, not pushed to the car

The Preferred SOC cap and Boost SOC cap are **local stop thresholds
inside this automation**. The controller reads the car's own
charge-limit number (the one you set in the Tesla app) but never
writes to it. Nothing about your Tesla configuration changes.

The effective stop point during a controlled solar session is:

```
effective_limit = min(local_cap, tesla_app_limit)
```

So if your Tesla app limit is 80 % and the Preferred SOC cap is
60 %, home solar charging stops at 60 %. A Supercharger session, a
non-controlled Wall Connector session, or any charging that happens
while the master toggle is off, will still charge up to the 80 %
car-side limit.

Practical consequence for the boost: to actually fill above the
Preferred cap, the car-side limit must be at or above the Boost
cap. Example: Tesla app 90 %, Preferred 60 %, Boost 80 % means
normal solar stops at 60 % and a boost day stops at 80 %.

## Forecast boost (optional)

The controller has two SOC caps:

| Input | Role |
|---|---|
| **Preferred SOC cap** | Normal stop point (e.g. 60 %). |
| **Boost SOC cap** | Higher stop point used only when the forecast says today is the last sunny day before bad weather (e.g. 80 %). |

The boost is **active for the day** when *all* of the following hold:

1. `solcast_today_kwh` is at or above the "sunny today" threshold
   (e.g. 25 kWh).
2. Every Solcast forecast for the next `boost_lookahead_days` (1 to
   6) is at or below the "bad day" threshold (e.g. 12 kWh), and
   every one of those sensors has a valid reading.
3. Boost SOC cap is greater than Preferred SOC cap.

If any upcoming forecast sensor is missing or unavailable, the boost
is suppressed (fail-safe: better to under-charge than to assume a
sunny week ahead based on partial data).

Leave all `solcast_*_kwh` inputs blank to disable the feature
entirely; the controller then behaves with `max_soc` as the only
cap.

> **Note on Solcast naming:** the upcoming-day sensors are
> `forecast_tomorrow`, `forecast_day_3`, `forecast_day_4`, ... so
> `day_3` is **two** days from today and `day_7` is **six** days
> from today. The lookahead consumes them in order, stopping at the
> first gap, so to look 3 days ahead set lookahead = 3 and provide
> tomorrow + day_3 + day_4.

## Charging window (dashboard control)

The controller takes the window start/end as `input_datetime` entity
inputs (time-only helpers). Create the helpers once, point the
controller at them, and add them to a dashboard card to change the
window without editing the automation:

```yaml
type: entities
title: Solar Charging
entities:
  - input_boolean.solar_charging_enabled
  - input_datetime.solar_window_start
  - input_datetime.solar_window_end
```

The controller re-evaluates every minute, so changes take effect
within ~60 s. Cross-midnight windows are not supported (string
comparison on `HH:MM:SS`).

## Electrical setup

Set these inputs on the controller to match your install:

| Install | `line_voltage` | `phase_count` |
|---|---|---|
| AU/EU 1-phase | 230 | 1 |
| AU/EU 3-phase | 230 | 3 |
| UK 1-phase | 240 | 1 |
| US split-phase (240 V) | 240 | 1 |
| US 1-phase (120 V) | 120 | 1 |

The grid export sensor must report **total** grid power across all
phases (standard for whole-house meters like Shelly EM, Enphase,
Powerwall). If your sensor reports per-phase watts instead, set
`phase_count` to 1.

Sign convention: the export sensor is **at or below 0 when
exporting** (positive when importing), and the import sensor is **at
or above 0 when importing**. They are combined additively inside
the controller so two-counter meters (Enphase-style: separate
unidirectional import and export registers) work correctly. If your
meter only exposes a single signed net sensor, point both controller
inputs at that same entity. The math still works because
`net + 0 == net`.

## Load prioritisation (optional)

The controller can pause other power-hungry automations while the
Tesla is low on charge, so the available solar export goes to the
car first. Wire each downstream automation behind its own
`input_boolean` (e.g. `input_boolean.ac_auto_enabled`,
`input_boolean.pool_pump_auto_enabled`) so the automation only runs
when its boolean is ON. Then point the controller's **Prioritize
Tesla over these loads** input at those booleans and pick a SOC
threshold (default 40 %).

Behaviour:

- While the charging window is open AND the Tesla SOC is below the
  threshold, the controller turns OFF any of the listed booleans
  that are currently ON.
- The controller turns ON any of those booleans that are currently
  OFF in two situations:
  1. As soon as the Tesla reaches the effective SOC cap (the lower
     of the local Preferred / Boost cap and the Tesla app limit).
     Restoring at this point means the loads come back online as
     soon as the car no longer needs the priority, not at the end
     of the window.
  2. At the window end time, as a safety net so loads can't get
     stuck off if the SOC cap is never reached (e.g. session ends
     for other reasons, or the car was unplugged).
- The window-end restore runs regardless of the master enable
  toggle so loads never get stuck off.
- The controller only writes to a boolean when its state would
  actually change. The logbook stays clean even though the tick
  branch evaluates every minute.
- The restore is unconditional: if you have your own reason to keep
  one of those automations paused, gate it from a different switch
  or pick a separate `input_boolean`.

Leave the input empty to disable the feature entirely.

## Runtime model

The consolidated controller is a single `mode: parallel` automation.
Each trigger carries an `id` and the action routes to a matching
branch:

| Trigger id | Source | Branch behaviour |
|---|---|---|
| `tick` | HA start, every minute, state changes on grid/charger/SOC | Main ramp / start / SOC-cap / window-end logic. Pauses prioritize-loads when SOC is low during the window, and restores them as soon as the SOC cap is reached. |
| `grace_expired` | Below-minimum flag held ON for the grace period | Stop the session and notify. |
| `ha_start_reconcile` | HA start | Clear a stale below-minimum flag if no session is running. |
| `import_spike` | Any change to the grid import sensor | Compute and apply a current trim if import exceeds the threshold. |
| `plugged_in` | Vehicle-connected goes ON | Notify; message depends on the enable toggle. |
| `enabled` | Enable toggle goes ON while plugged in | Notify. |
| `disabled` | Enable toggle goes OFF mid-session | Notify. |
| `window_close` | Time-of-day equal to the window end helper | Restore any prioritize-loads currently off. |

Parallel mode lets a fast import trim run while a slow ramp is still
in its stability delay. The trim is purely a ramp-down; the controller
remains the single source of truth for stop decisions.

## License

MIT
