# Solar-Aware Tesla Charging (Home Assistant Blueprint)

Charges a Tesla on a Wall Connector from your solar surplus only, by
tracking household consumption and PV output minute by minute.

**Why use it.** The Tesla app and most third-party schedulers force a
single static charge current. As soon as a cloud passes or an
appliance turns on, that fixed setpoint is wrong. Set it high and
the car pulls from the grid whenever solar dips below the
setpoint. Set it low to avoid imports and you instead export the
rest of your solar at the low feed-in tariff because the car can't
absorb it. Either way you're paying for it: the gap between import
and feed-in tariffs is where the money goes. This blueprint instead
matches the car's charge current to your real-time solar export, so
the session runs on the solar your house isn't already consuming.
The practical effect is more of your own generation ending up in
the car instead of being exported. In markets where the import
tariff is meaningfully higher than the feed-in tariff (most of
Australia, the UK, much of Europe), this also reduces your
electricity bill: every kWh kept on-site is a kWh you don't buy
back later at retail.

**How it works.** A single Home Assistant automation reads your
existing grid import and export sensors (Enphase, Shelly EM,
Powerwall, Emporia, generic CT clamps via ESPHome, etc.) and
modulates the Tesla's charge current through the official Tesla
integration. It uses the Fleet API's 1 A floor instead of the Tesla
app's 5 A floor, ramps continuously between 1 A (~700 W on 3-phase)
and the Wall Connector's ~11 kW ceiling, and reacts to grid state
changes within seconds rather than at the next minute boundary.

## Features

- Ramps current between 1 A and charger max. Never stop/starts unnecessarily.
- Holds at minimum through short cloud gaps before ending a session,
  with a configurable grace period; if export recovers, the grace
  timer aborts automatically.
- Instant grid-import guard: trims current the moment any appliance
  causes net import.
- Fast recovery after a brief import spike. After the guard trims,
  the main ramp loop skips its stability delay for a few minutes,
  so a kettle pulse doesn't pin charging low for the rest of the
  session.
- **Fast ramp-up** (configurable): when available export jumps well
  ahead of the current setpoint (default 3 A of headroom), jump
  directly to target in one tick instead of the conservative +1 A
  per minute. Recovers from a cloud-clearing event in one step.
- Uses Tesla Fleet API's 1 A floor (about 690 W on 3-phase), not the
  app's 5 A.
- Works with 1-, 2-, or 3-phase installs (configurable line voltage
  and phase count).
- SOC-aware: stops at the lower of the configured cap and the Tesla
  car-side charge limit. The car-side limit is read but never
  written, so your Tesla app setting is left untouched.
- Lifecycle notifications: car plugged in (whether the master toggle
  is on or off), master toggle flipped on while plugged in, master
  toggle flipped off mid-session.
- Session-end notifications for SOC cap, optional hard window-end
  stop, and solar-exhaustion stop.
- Wakes the Tesla automatically when its status is asleep or unknown
  at the moment the controller tries to restart charging, so a
  mid-session car sleep doesn't block the resume.
- Optional **Solcast forecast boost**: raises the SOC cap on a sunny
  day when the next 1 to 6 days are forecast to be poor, so you
  stash extra charge before bad weather. Disabled by default;
  activates only when Solcast sensors are configured. Suppressed
  if any required forecast sensor is missing or unavailable.
- Optional **load prioritisation**: temporarily pauses other
  power-hungry automations (AC, pool pump, etc.) while the Tesla
  SOC is low so the car gets first claim on solar export. Loads
  are restored automatically when the SOC cap is reached or when
  the vehicle is unplugged, with a window-end safety net.
- Optional **deferrable load awareness**: point the controller at
  power sensors for solar diverters (e.g. a myenergi eddi) and
  their consumption is treated as available surplus. The Tesla
  ramps up to reclaim it, the diverter backs off naturally, and
  brief ramp-up overshoots are not punished by the import-spike
  trim.
- Optional **session start lock**: dedupes "Solar Charging Started"
  notifications when the start logic re-enters during the 15 s
  car-wake delay. Auto-clears on contactor open, HA restart, or
  unplug.
- Optional **Tesla-unreachable guard**: when Tesla Fleet API
  entities are unavailable, the controller skips its per-minute
  evaluation rather than logging service-call errors. Resumes
  automatically as soon as the API recovers.
- **Restart SOC hysteresis**: after stopping at the cap, won't
  restart until SOC has drained back below `cap - hysteresis`,
  so vampire drain can't trigger tiny post-cap relaunches.
- **Start verification**: after turning the switch on, waits for
  the contactor to actually close. A silent Fleet API drop
  releases the session locks and skips the "Started" notification
  so the next tick can retry cleanly.
- **Mid-session fault stop**: if the wall connector reports
  `fault` / `faulted` / `error` / `offline` while charging, the
  controller stops and notifies.
- Optional **ramp-up rate limiter** (`input_datetime` helper):
  properly serialises ramp-up across the parallel-execution slots
  by gating on elapsed time, fixing the case where new state-
  change triggers bypass the in-action stability delay.
- Optional **boost session lock**: snapshots the Solcast boost at
  session start so the cap doesn't drop below live SOC mid-day
  if the forecast flips.
- Optional **grid top-up safety net**: a deadline + floor pair
  that falls back to fixed-current grid charging when prolonged
  poor weather keeps the car under the floor.
- Optional **home battery awareness**: while the house battery is
  below its reserve SOC, the Tesla controller treats export as
  zero so the inverter charges the battery first.
- **Notification dedupe**: session-end notifications are
  suppressed if the session didn't move SOC by at least a
  configurable amount, so cloudy-day cycles stay quiet.
- Robust against missing or unavailable sensors throughout. Each
  branch checks its inputs first and skips silently rather than
  running on bad data.

## Blueprints

| Blueprint | Purpose |
|---|---|
| `solar_tesla_controller.yaml` | Single all-in-one automation. Controls ramp / start / stop, runs the import-spike trim, the below-minimum grace stop, the plug/toggle lifecycle notifications, load prioritisation, and session-end cleanup. |

Earlier versions of this repo split the behaviour across four
blueprints (Controller + Grace Stop + Import Guard + Plugged-In
Notify), plus a separate Stop-At-SOC blueprint. The consolidated
controller absorbs all of them. If you're upgrading, delete the
retired blueprints and the automations they powered, then import
the new controller and configure it once.

## Import

In Home Assistant: **Settings -> Automations -> Blueprints -> Import
Blueprint**, then paste the raw GitHub URL of
`solar_tesla_controller.yaml`.

### Pinning to a release

For a stable install that won't change unexpectedly, import from a
tagged release instead of `main`:

```
https://github.com/andre-sam/solar-tesla-charging/blob/v1.0.0/solar_tesla_controller.yaml
```

The [Releases page](https://github.com/andre-sam/solar-tesla-charging/releases)
lists every version. Use the URL from a release tag to pin; use
the `main` URL to always track the latest (may include in-progress
changes).

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
- A third **`input_boolean`** as a session start lock (e.g.
  `input_boolean.solar_charging_session_active`). Dedupes the
  "Solar Charging Started" notification when the start logic
  re-enters during the 15 s wake delay. See [Session start lock](#session-start-lock-optional).
- One or more **`input_boolean`** entities that gate other power-
  hungry automations (AC, pool pump, etc.) you want the controller
  to pause while the Tesla SOC is low. See [Load prioritisation](#load-prioritisation-optional).
- One or more **power sensors** (W or kW) for deferrable loads
  (e.g. `sensor.myenergi_eddi_internal_load_ct1`) that you'd
  rather have the Tesla outbid. See [Deferrable load awareness](#deferrable-load-awareness-optional).
- An **`input_number`** (0 to 100, step 1) as a session start SOC
  tracker (e.g. `input_number.solar_charging_start_soc`). The
  controller writes the car's SOC to it at the start of each
  charging session and reads it back at session end to append a
  "charged from X% (+Y%)" summary to stop notifications. You don't
  interact with it directly. Leave the input empty to skip the
  summary suffix.
- An **`input_datetime`** (date + time) as a last-ramp-up timestamp
  (e.g. `input_datetime.solar_charging_last_ramp_up`). When set,
  the controller properly rate-limits Branch 6 ramp-up steps across
  parallel-execution slots using the stability delay. Without this
  helper, the in-action delay only single-shots the originating
  trigger; new state-change ticks can still spawn fresh ramps. See
  [Ramp-up rate limiting](#ramp-up-rate-limiting-optional).
- An **`input_boolean`** as a boost session lock (e.g.
  `input_boolean.solar_charging_boost_active`). Latched ON at the
  start of a session if the Solcast boost is live at that moment;
  keeps the Boost SOC cap in force for the whole session even if
  the live forecast flips off mid-day. Cleared automatically on
  session end. Only meaningful when the forecast boost is also
  configured.
- A **home battery SOC sensor** plus a reserve threshold to make
  the controller yield to the house battery while it's below the
  reserve. See [Home battery awareness](#home-battery-awareness-optional).
- An **`input_datetime`** (time only) plus a floor SOC to enable a
  grid top-up safety net for prolonged poor weather. See
  [Grid top-up safety net](#grid-top-up-safety-net-optional).
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

The effective stop point while this automation is in control of the
session is:

```
effective_limit = min(local_cap, tesla_app_limit)
```

So if your Tesla app limit is 80 % and the Preferred SOC cap is
60 %, home solar charging stops at 60 %. A Supercharger session, a
Wall Connector session started while the master toggle is OFF, or
any other charging not driven by this automation, will still charge
up to the 80 % car-side limit.

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
within ~60 s. Cross-midnight windows are supported: if the end
time is earlier than the start time (e.g. start 22:00, end 06:00)
the window wraps midnight and is treated as open from start
through end on the following day.

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
  that are currently ON. If a notify service is configured, a
  single "Loads Paused for Tesla" notification is sent listing
  which loads were just paused.
- The controller turns ON any of those booleans that are currently
  OFF in three situations:
  1. As soon as the Tesla reaches the effective SOC cap (the lower
     of the local Preferred / Boost cap and the Tesla app limit).
     Restoring at this point means the loads come back online as
     soon as the car no longer needs the priority, not at the end
     of the window.
  2. As soon as the vehicle is unplugged from the wall connector,
     so loads like AC come back on immediately when the car leaves
     mid-window rather than waiting for the window to end.
  3. At the window end time, as a safety net so loads can't get
     stuck off if neither of the above has happened (e.g. session
     ends for other reasons while the car stays plugged in).
- The unplug and window-end restores both run regardless of the
  master enable toggle so loads never get stuck off.
- The controller only writes to a boolean when its state would
  actually change, so the logbook stays clean even though the
  controller re-evaluates every minute.
- The restore turns each boolean back ON unconditionally; the
  controller doesn't track *why* it was off. If you have your own
  reason to keep one of those automations paused, gate it from a
  different switch so the controller's restore can't undo your
  intent.

Leave the input empty to disable the feature entirely.

## Deferrable load awareness (optional)

If you have a solar diverter (myenergi eddi, Solic 200, custom
ESPHome script that PWMs an immersion heater, etc.) the controller
can be made aware of it so the Tesla doesn't "see" all your surplus
as already taken.

The diverter modulates to soak up whatever surplus is left after the
rest of the house. From the Tesla controller's perspective, that
makes the grid look perfectly balanced and there is nothing to
claim. Wire a power sensor for the diverter's load into the
controller's **Deferrable load power sensors** input and that
consumption is added to the available export budget. The Tesla
ramps up, the export shrinks, the diverter sees less surplus and
backs off on its own. Net result: the Tesla gets first claim on PV,
the diverter still gets whatever the Tesla can't use, no on/off
handshake required.

How it's computed:

- Only the **solar-fed portion** of the deferrable load is added
  back: `reclaimable = max(0, deferrable_load_w - grid_import_w)`.
  This caps headroom at the load that is actually being supplied
  by solar, so the controller never claims grid-imported power as
  surplus.
- The import-spike trim treats import that is still covered by the
  deferrable load as expected ramp-up overshoot and skips the
  trim. If the load has fully backed off and the import persists,
  the trim runs as normal on whatever import is left over.
- Multiple sensors are summed. Units are honoured per sensor (W or
  kW). Unavailable / unknown sensors are skipped silently.
- Negative readings are ignored (the input is meant for
  consumption-only sensors).

When NOT to use this:

- A non-deferrable load that just happens to be on (kettle, oven).
  Those won't back off when you crowd them out, you'll just import.
  Only point this input at loads you trust to yield on their own.
- A diverter whose output is already netted out of your grid export
  sensor by the meter (some installs have the diverter wired
  upstream of the CT). In that case the diverter is already invisible
  to the controller in the way you want, and adding the sensor here
  would double-count.

If your diverter has a very slow response (more than ~10 s to back
off), expect short import bursts during Tesla ramp-up. The
import-spike trim absorbs the worst of it, but raising
`import_safety_margin_amps` by 1 can help.

Leave the input empty to disable the feature entirely.

## Session start lock (optional)

The controller runs in parallel mode so a fast import trim can
interrupt a slow ramp. As a side-effect, the start logic can fire
a second time while the previous run is still in its 15 s car-wake
delay, producing a duplicate "Solar Charging Started"
notification. The session start lock prevents this.

Wire-up:

1. Create an `input_boolean` helper (e.g.
   `input_boolean.solar_charging_session_active`).
2. Point the controller's **Session start lock (optional)** input
   at it.
3. Done. The controller turns it ON at the start of a session and
   OFF when the contactor opens for 10 seconds, on HA restart
   (when no session is live), when the car is unplugged, or when
   the master enable toggle is turned off before the contactor
   actually closed (e.g. during the 15 s wake delay).

Leave the input empty to skip the lock; duplicate "Started"
notifications are then possible if the start logic re-enters
during the wake delay.

## Restart hysteresis

After Branch 1 stops a session at the effective SOC cap, the
Tesla idles plugged in and vampire drain slowly bleeds the SOC
back below the cap. Without hysteresis the controller would
restart a fresh session as soon as SOC dropped to `cap - 0.1`,
contributing nothing useful and producing notification storms.

The **Restart SOC hysteresis (%)** input (default 2) bars Branch
5 from restarting until SOC has fallen at least that far below
the effective cap. Set to 0 to restore the old behaviour.

## Ramp-up rate limiting (optional)

The controller runs in `mode: parallel, max: 10` so a fast
import trim can pre-empt a slow ramp. As a side-effect, the
in-action `delay` in Branch 6 only single-shots the originating
run — any new state-change tick spawns a fresh parallel run that
re-evaluates and writes immediately, bypassing the stability
delay.

Configure the optional **Last ramp-up timestamp** input
(`input_datetime` with date + time, e.g.
`input_datetime.solar_charging_last_ramp_up`) and the controller
stamps it on every raise. Subsequent ramp-up attempts in any
parallel slot then check elapsed time against
`stability_delay_seconds` and skip if not enough time has passed.
The post-trim fast-recovery window still bypasses the gate for
quick cloud-clearing recovery.

Without this helper, ramp-down is unaffected and ramp-up still
works — it just isn't strictly rate-limited.

## Start verification

After turning the charge switch on, the controller now waits up
to **Start verification timeout (s)** (default 60) for the
contactor to actually close. If it doesn't (silent Fleet API
drop, car wedged mid-wake, etc.), the controller:

- Releases the session start lock and the boost session lock so
  a future tick can retry.
- Suppresses the "Solar Charging Started" notification so you
  don't get a "started" message for a session that never
  actually started.

This eliminates the failure mode where a stuck `session_lock`
blocked further restart attempts until HA restart.

## Notification dedupe by SOC gain

Cloudy days used to produce a stream of "Started" → "Session
Ended" notifications when the controller cycled the session
multiple times without meaningful progress. The
**Minimum SOC gain for session-end notifications (%)** input
(default 1) suppresses the grace-stop, SOC-cap, window-close,
and grid-top-up notifications when SOC didn't move by at least
that much during the session. Set to 0 to always notify.

## Grid top-up safety net (optional)

For prolonged poor weather, you can configure a deadline + floor
so the controller falls back to grid charging:

| Input | Role |
|---|---|
| **Minimum SOC floor (%)** | If SOC is below this at the deadline, start a grid session. Set to 0 to disable. |
| **Minimum SOC deadline** (`input_datetime`, time only) | When (within the charging window) to start the top-up if the car is still below the floor. |
| **Grid top-up current (A)** | Fixed current used during top-up (e.g. 8 A ≈ 1.8 kW on 230 V single-phase). |

Behaviour:

- Branch 5b starts a session at the configured top-up current
  even when there is no solar export. Notifies "Grid Top-Up
  Started".
- Branch 5c stops the session as soon as SOC reaches the floor
  (without waiting for the effective SOC cap or grace timer).
  Notifies "Grid Top-Up Complete" (subject to the gain dedupe
  above).
- Honours cross-midnight charging windows.
- Capped at the Tesla app limit; never charges above it.

Leave the deadline input or the floor empty (`0`) to disable the
safety net entirely. The default behaviour is unchanged.

## Home battery awareness (optional)

On installs with a hybrid inverter and a house battery (Powerwall,
Sungrow SBR, BYD, Sigenergy, etc.), you can have the Tesla yield
to the house battery while it is still topping up:

| Input | Role |
|---|---|
| **Home battery SOC sensor (%)** | Battery state-of-charge sensor. Leave empty to disable. |
| **Home battery reserve SOC (%)** | While the battery is below this SOC, the Tesla controller treats available export as zero. Set to 0 to disable (Tesla competes normally). |

The export sensor signal is unchanged; the controller just stops
claiming export below the reserve so the inverter sends surplus
to the battery. Once the battery hits the reserve, the Tesla
ramps up normally. Fail-open: if the sensor is unavailable, the
gate is treated as not configured (so a flaky sensor can't
silently prevent the Tesla from ever charging).

## Tuning for cloudy / fluctuating solar

The defaults are tuned for stable solar with a generous buffer.
If your sky is patchy, the following inputs make the controller
track solar more aggressively at the cost of more Fleet API
chatter and slightly higher chance of brief grid imports:

| Input | Default | Aggressive value | Effect |
|---|---|---|---|
| `export_threshold_amps` | 1 | 0 | Removes the constant ~230 W (1-ph) / 690 W (3-ph) export buffer. |
| `stability_delay_seconds` | 60 | 15 to 30 | Time the controller waits between ramp-up steps. |
| `post_trim_fast_recovery_seconds` | 180 | 300 to 600 | After an import-spike trim, the stability delay is skipped for this long, so the controller keeps tracking solar without the per-step pause through a cloud event. |
| `import_threshold_w` | 50 | 25 to 30 | Noise floor for the import-spike trim. Lower = reacts to smaller imports. |
| `fast_ramp_headroom_amps` | 3 | 2 | When available export is at least this many amps above the current setpoint, jump directly to target instead of the +1 A per minute cap. |

If you also have a fine-grained PV diverter (myenergi Eddi or
similar) on the same circuit, leave it to handle the residual
below ~700 W. Tesla can't compete with phase-angle PWM control at
sub-amp resolution. You can also point the controller's
**Deferrable load power sensors** input at the diverter's CT so
the Tesla actively reclaims the diverter's bigger draws; see
[Deferrable load awareness](#deferrable-load-awareness-optional).

## Runtime model

The controller is one automation with several triggers. Each
trigger carries an `id` and the action routes to a matching
branch. Triggers run in parallel so a fast import trim can
execute alongside a slow ramp.

| Trigger id | Source | Branch behaviour |
|---|---|---|
| `tick` | HA start, every minute, state changes on grid/charger/SOC | Main ramp / start / SOC-cap / window-end logic, including the mid-session wall-connector fault stop and the grid top-up start/stop. Pauses prioritize-loads when SOC is low during the window, and restores them as soon as the SOC cap is reached. Adds any configured deferrable load consumption (capped at the solar-fed portion) to the available export budget. Treats available export as zero while a configured home battery is below its reserve. Skipped when Tesla Fleet API entities are unavailable. |
| `grace_expired` | Below-minimum flag held ON for the grace period | Stop the session, clear locks, and notify (subject to the SOC-gain dedupe). |
| `ha_start_reconcile` | HA start | Clear stale stateful flags (below-minimum tracker, session start lock, boost session lock) if no session is running. |
| `import_spike` | Grid import crosses above `import_threshold_w` | Compute and apply a current trim if import exceeds the threshold. Import covered by configured deferrable loads is treated as expected ramp-up overshoot and skipped. The numeric-state edge trigger replaces the old per-change trigger so sub-threshold meter chatter doesn't fill the parallel-execution slot pool. |
| `plugged_in` | Vehicle-connected goes ON | Notify; message depends on the enable toggle. |
| `enabled` | Enable toggle goes ON while plugged in | Notify. |
| `disabled` | Enable toggle goes OFF mid-session | Stop the charger (turn off `charge_switch`, clear `below_min_flag`, restore prioritize-loads) and notify. Also clears the session and boost locks if the contactor never closed. |
| `unplugged` | Vehicle-connected goes OFF for 10 s | Restore any prioritize-loads currently off so loads like AC come back on as soon as the car leaves the charger. Also clears the session and boost locks if claimed but the contactor never closed. |
| `window_close` | Time-of-day equal to the window end helper | Restore any prioritize-loads currently off. |
| `session_ended` | Contactor goes from ON to OFF for 10 s | Clear the session and boost locks so a future start can fire. |

The import trim only ever ramps current down; only the per-minute
`tick` branch can stop or restart a session, so there's no risk of
the two branches fighting each other. Branch 5 ("Start / resume")
and Branch 5b ("Grid top-up start") both verify the contactor
actually closes after the switch is turned on, and roll back the
locks plus suppress the "Started" notification if it doesn't.

## Troubleshooting

Each item is **Symptom → Likely cause → Fix**.

**Tesla never starts charging even on a sunny day.**
The export sensor sign is wrong. The blueprint expects
`grid_export_power_negative` to read **below 0 W when exporting**.
Quick check: while you're actively pushing power to the grid, look
at the configured sensor — if it's a positive number, your
underlying meter reports export as positive, not negative. Wrap it
in a template sensor that multiplies by `-1`, or pick the correct
signed register from your meter integration.

**Tesla overshoots and imports from the grid on a 3-phase install.**
`phase_count` mismatch. The blueprint assumes your grid sensor
reports **total** grid power across all phases (typical for
whole-house meters like Shelly EM, Enphase, Powerwall). If your
sensor reports per-phase watts, set `phase_count` to 1 — the math
already accounts for "watts per amp = line_voltage × phase_count"
and a per-phase reading is effectively single-phase from the
controller's perspective.

**Deferrable load awareness double-counts; every ramp-up causes a
brief grid import.** The diverter is wired *upstream* of the grid
CT, so its consumption is already netted out of your export
reading. Adding the diverter's sensor to **Deferrable load power
sensors** then credits it a second time. Fix: leave that input
empty for upstream-wired diverters.

**Charging stops at a lower SOC than the Preferred cap.** The
Tesla app charge limit is below the Preferred cap. The effective
stop is `min(local_cap, car_side_limit)`. Either raise the limit
in the Tesla app or accept the lower stop point.

**Charging stops at the Preferred cap even on a "boost" day.**
The Tesla app limit is below the Boost cap, so the boost is gated
by the car-side limit. Raise the Tesla app limit to at least the
Boost cap if you want forecast-driven boosting to actually fill
higher.

**Boost dropped mid-day and the session ended at the Preferred cap
instead of the Boost cap.** The optional **Boost session lock**
helper isn't configured. Without it, a brief Solcast unavailable
flips `boost_active` off mid-day; the cap drops from Boost to
Preferred, and if the current SOC is already above Preferred,
Branch 1 fires immediately. Fix: create a dedicated
`input_boolean` (e.g. `input_boolean.solar_charging_boost_active`)
and configure it in the **Boost session lock** input. The
controller latches it ON at session start and ignores the live
forecast until the session ends.

**`max_charge_amps` setting appears ignored.** The controller
clamps against the Tesla `number` entity's own `max` attribute. If
the Tesla integration exposes the entity with `max: 16` and you've
set 32 A in the blueprint, you'll get 16 A. This is intentional —
the integration's max is the authoritative ceiling.

**"+X%" delta in the end-of-session notification is smaller than
expected.** `session_start_soc` is overwritten on every Branch 5
entry, so the "+X%" is measured from the **last contactor close**,
not the start of the day. If the session stopped and restarted
during cloud cover (e.g. via grace stop + resume), the delta only
covers the final segment. Documented behaviour.

**"Solar Charging Started" notification fires but the car doesn't
actually charge.** Tesla Fleet API call was accepted but the car
was wedged or the integration dropped the request silently. With
**Start verification timeout (s)** (default 60), the controller
now suppresses this notification when `contactor_closed` doesn't
flip to ON within that window. If you're still seeing premature
"Started" messages: check that the wall-connector
`contactor_closed` binary sensor actually responds when current is
flowing (some integrations expose only the `status` text sensor
reliably).

**Notifications went quiet during cloudy weather.** Working as
intended. **Minimum SOC gain for session-end notifications (%)**
(default 1) suppresses the grace-stop / SOC-cap / window-close
notifications when SOC didn't move by at least that much. Set to
0 if you want every cycle reported.

**Tesla wakes but charging won't start.** The car's status is
probably `unknown` and the wake button either isn't configured or
didn't connect. Verify the **Tesla wake-up button** input is set
to the integration's wake `button.*` entity. Cold cars and cars
in deep sleep occasionally need longer than the built-in 15 s
wake delay; if you see this regularly, raising the delay would
require a code change rather than a config tweak.

**Solcast forecast boost never activates even though tomorrow looks
bad.** Solcast's sensor naming is off by one: `forecast_day_3` is
**+2 days from today**, not +3. Configure today + tomorrow +
day_3 + day_4 if you want a 3-day lookahead. The README's
[Forecast boost](#forecast-boost-optional) section has a note on
this, but it's still the most common Solcast configuration
mistake.

## License

MIT. Copyright (c) 2024-2026 Andre Sambade. See [LICENSE](LICENSE) for
the full text.
