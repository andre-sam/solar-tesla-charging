# Solar-Aware Tesla Charging (Home Assistant Blueprints)

Ramp-first solar charging control for a Tesla Wall Connector using any
grid-power sensor pair (Enphase, Shelly EM, Powerwall, etc.).

- Ramps current between 1 A and charger max — never stop/starts unnecessarily.
- Holds at minimum through short cloud gaps before ending a session.
- Separate grid-import guard trims current instantly if any appliance kicks in.
- Fast recovery after a brief import spike: the Guard can stamp an
  `input_datetime` the Controller reads to skip its stability delay.
- Uses Tesla Fleet API's 1 A floor (≈ 690 W on 3-phase), not the app's 5 A.
- Works with 1-, 2-, or 3-phase installs (configurable line voltage and phase count).
- SOC-aware: the Controller is the single source of truth for the SOC
  cap stop (optional notify).
- Session end notifications for SOC-cap, window-end (optional hard
  stop) and solar-exhaustion paths.
- Wakes an ambiguous-status Tesla on resume so a mid-session sleep
  doesn't block restart.
- Optional **Solcast forecast boost**: raises the SOC cap on a sunny
  day when the next 1–3 days are forecast to be poor, so you stash
  extra charge before bad weather. Disabled by default; activates only
  when Solcast sensors are configured.
- Grace-period stop lives in a separate companion automation so the
  Controller stays responsive: if export recovers, the timer aborts.

## Blueprints

The **Controller** + **Grace Stop** pair is the minimum viable install.
The others are optional enhancements.

| Blueprint | Purpose |
|---|---|
| `solar_tesla_controller.yaml` | Main ramp-up / ramp-down / start / resume / SOC-cap stop. Toggles the below-minimum flag. |
| `solar_tesla_grace_stop.yaml` | Ends the session after the below-minimum flag has stayed on for the grace period. |
| `solar_tesla_import_guard.yaml` | Instant current cut on any grid import. Optionally stamps a trim timestamp the Controller reads for fast recovery. |
| `solar_tesla_plugged_in_notify.yaml` | Notifies when plugged in (whether solar charging is off or on) and when the toggle is enabled while already plugged in. |
| `solar_tesla_stop_at_soc.yaml` | **Deprecated.** The Controller now handles the SOC cap stop itself. Kept only for users who already imported it. |

## Import

In Home Assistant: **Settings → Automations → Blueprints → Import Blueprint**,
then paste the raw GitHub URL of each YAML you want.

## Required entities

You need these from your installation (any integration, any names):

- A **grid export power** sensor in W (≤ 0 when exporting — signed net or
  dedicated export counter)
- A **grid import power** sensor in W (≥ 0 when importing — dedicated
  import counter, or the same signed sensor as above)
- A **wall connector power** sensor in W or kW
- A **wall connector vehicle-connected** binary sensor
- A **wall connector contactor-closed** binary sensor
- A **Tesla charge current** `number` entity (amps)
- A **Tesla charge switch** (on/off)
- A **Tesla battery SOC** sensor (%)
- An **`input_boolean`** you create yourself as the master enable toggle
  (e.g. `input_boolean.solar_charging_enabled`)
- A second **`input_boolean`** for the below-minimum tracker shared
  between the Controller and the Grace Stop blueprint
  (e.g. `input_boolean.solar_charging_below_min`)
- Two **`input_datetime`** helpers (time-only) for the charging window,
  shared between the Controller and the Plugged-In Notify blueprint
  (e.g. `input_datetime.solar_window_start`,
  `input_datetime.solar_window_end`). Expose these on your dashboard to
  change the window from the EV view — both blueprints read the same
  helpers, so notifications and control stay in sync.

Optional:

- A **Tesla charge limit** `number` entity (blueprint falls back to the
  configured SOC cap if absent)
- A **wake-up** `button` for the Tesla integration
- An **`input_datetime`** (date + time) shared between the Import Guard
  and the Controller to enable fast recovery after brief import spikes
  (e.g. `input_datetime.solar_charging_last_guard_trim`)
- A **notify service** (e.g. `notify.mobile_app_phone`) for session
  notifications (SOC reached, window closed, solar-exhaustion stop)
- **Solcast PV Forecast** daily-total sensors for the forecast boost.
  Pick today, tomorrow, and as many of `day_3`–`day_7` as you want
  (the lookahead input chooses how far ahead to inspect):
  `sensor.solcast_pv_forecast_forecast_today`,
  `sensor.solcast_pv_forecast_forecast_tomorrow`,
  `sensor.solcast_pv_forecast_forecast_day_3`,
  `sensor.solcast_pv_forecast_forecast_day_4`,
  `sensor.solcast_pv_forecast_forecast_day_5`,
  `sensor.solcast_pv_forecast_forecast_day_6`,
  `sensor.solcast_pv_forecast_forecast_day_7`

## Forecast boost (optional)

The Controller has two SOC caps:

| Input | Role |
|---|---|
| **Preferred SOC cap** | Normal stop point (e.g. 60 %). |
| **Boost SOC cap** | Higher stop point used only when the forecast says today is the last sunny day before bad weather (e.g. 80 %). |

The boost is **active for the day** when *all* of the following hold:

1. `solcast_today_kwh` ≥ "sunny today" threshold (e.g. 25 kWh).
2. Every Solcast forecast for the next `boost_lookahead_days` (1–6) is
   ≤ "bad day" threshold (e.g. 12 kWh) — and every one of those
   sensors has a valid reading.
3. Boost SOC cap > Preferred SOC cap.

If any upcoming forecast sensor is missing or unavailable, the boost
is suppressed (fail-safe — better to under-charge than to assume a
sunny week ahead based on partial data).

Leave all `solcast_*_kwh` inputs blank to disable the feature
entirely; the Controller then behaves exactly as before with
`max_soc` as the only cap.

> **Note on Solcast naming:** the upcoming-day sensors are
> `forecast_tomorrow`, `forecast_day_3`, `forecast_day_4`, … so
> `day_3` is **two** days from today, `day_7` is **six** days from
> today. The lookahead consumes them in order, so to look 3 days
> ahead set lookahead = 3 and provide tomorrow + day_3 + day_4.

## Charging window (dashboard control)

The Controller and Plugged-In Notify blueprints both take the window
start/end as `input_datetime` entity inputs (time-only helpers). Create
the helpers once, point both blueprints at them, and add them to a
dashboard card to change the window without editing automations:

```yaml
type: entities
title: Solar Charging
entities:
  - input_boolean.solar_charging_enabled
  - input_datetime.solar_window_start
  - input_datetime.solar_window_end
```

The Controller re-evaluates every minute, so changes take effect within
~60 s. Cross-midnight windows are not supported (string comparison on
`HH:MM:SS`).

## Electrical setup

Set these inputs on the Controller to match your install:

| Install | `line_voltage` | `phase_count` |
|---|---|---|
| AU/EU 1-phase | 230 | 1 |
| AU/EU 3-phase | 230 | 3 |
| UK 1-phase | 240 | 1 |
| US split-phase (240 V) | 240 | 1 |
| US 1-phase (120 V) | 120 | 1 |

The grid export sensor must report **total** grid power across all phases
(standard for whole-house meters like Shelly EM, Enphase, Powerwall). If
your sensor reports per-phase watts instead, set `phase_count` to 1.

Sign convention: the export sensor is **≤ 0 when exporting** (positive
when importing), and the import sensor is **≥ 0 when importing**. They
are combined additively inside the controller so two-counter meters
(Enphase-style: separate unidirectional import and export registers)
work correctly. If your meter only exposes a single signed net sensor,
point both controller inputs at that same entity — the math still works
because `net + 0 == net`.

## License

MIT
