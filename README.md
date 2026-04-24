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

Optional:

- A **Tesla charge limit** `number` entity (blueprint falls back to the
  configured SOC cap if absent)
- A **wake-up** `button` for the Tesla integration
- An **`input_datetime`** (date + time) shared between the Import Guard
  and the Controller to enable fast recovery after brief import spikes
  (e.g. `input_datetime.solar_charging_last_guard_trim`)
- A **notify service** (e.g. `notify.mobile_app_phone`) for session
  notifications (SOC reached, window closed, solar-exhaustion stop)

## Electrical setup

Set these inputs on the Controller to match your install:

| Install | `line_voltage` | `phase_count` |
|---|---|---|
| EU 1-phase | 230 | 1 |
| EU 3-phase | 230 | 3 |
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
