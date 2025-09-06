## 1) Fork plan & structure

Create these top‑level additions:

```
  __init__.py
  manifest.json
  config_flow.py
  coordinator.py
  api.py                # wrapper around pyEight; retry/backoff
  alarm.py              # NEW: alarm CRUD + one-off
  services.yaml         # NEW services (see below)
  climate.py            # map -100..100 to °C/°F view; forward to API levels
  number.py             # includes head/feet angles (existing) + rate-limits
  select.py             # base presets, bed side, autopilot mode
  binary_sensor.py      # presence, snore mitigation active
  sensor.py             # HR, HRV, RR, sleep scores, room temp, etc.
  event.py              # NEW: fire HA events (alarm_started, priming_done, snore_detected)
  translations/...
blueprints/automation/luca/
  eight_sleep_smart_priming.yaml
  eight_sleep_wakeup_routine.yaml
  eight_sleep_autopilot_loop.yaml
dashboards/eight_sleep/ (view YAML & resources)
scripts/eight_sleep/ (helpers called by automations)
docs/...
```


---

## 2) Code upgrades (services/entities/events)

Below are targeted diffs/snippets you can apply. I’m aligning to the existing code style (DataUpdateCoordinator, async I/O, services defined in `services.yaml`, translations, etc.). Where file names differ on your branch, keep the logic and signatures.

### 2.1 `services.yaml` (add/edit)

```yaml
# custom_components/eight_sleep_plus/services.yaml
set_smart_alarm:
  name: Set Smart Alarm (Recurring)
  description: Create/update a recurring alarm for the selected side.
  target:
    entity:
      domain: sensor
  fields:
    side:
      selector: { select: { options: ["left","right"] } }
      required: true
      description: Bed side
    time:
      selector: { time: {} }
      required: true
      description: Alarm time (HH:MM)
    weekdays:
      selector: { select: { multiple: true, options: ["monday","tuesday","wednesday","thursday","friday","saturday","sunday"] } }
      required: true
    vibration_power:
      selector: { number: { min: 0, max: 100, step: 5, mode: slider } }
      required: false
    thermal_wake:
      selector: { boolean: {} }
      required: false
    smart_light_sleep:
      selector: { boolean: {} }
      required: false

set_one_off_alarm:
  name: Set One-Off Alarm
  description: Create a one-off alarm for the next wake.
  target:
    entity:
      domain: sensor
  fields:
    side: { selector: { select: { options: ["left","right"] } }, required: true }
    datetime:
      selector: { datetime: {} }
      required: true
    vibration_power: { selector: { number: { min: 0, max: 100, step: 5 } } }
    thermal_wake: { selector: { boolean: {} } }
    smart_light_sleep: { selector: { boolean: {} } }

edit_bedtime_schedule:
  name: Edit Bedtime Schedule
  description: Set bedtime and per-stage target levels (API levels -100..100)
  target: { entity: { domain: sensor } }
  fields:
    side: { selector: { select: { options: ["left","right"] } }, required: true }
    bedtime: { selector: { time: {} }, required: true }
    bedtime_level: { selector: { number: { min: -100, max: 100, step: 1 } } }
    initial_level: { selector: { number: { min: -100, max: 100, step: 1 } } }
    final_level: { selector: { number: { min: -100, max: 100, step: 1 } } }

autopilot_set_mode:
  name: Autopilot+ Mode
  description: Set the Autopilot+ controller policy
  target: { entity: { domain: select } }
  fields:
    mode: { selector: { select: { options: ["off","conservative","balanced","aggressive"] } }, required: true }

# Keep existing services: heat_set, heat_increment, side_on, side_off, prime_pod, set_bed_side, start_away, stop_away, alarm_snooze, alarm_stop, alarm_dismiss.
```

> Beta releases mention “Add routine editing” and “Add one off alarms”. The signatures above map cleanly to `pyEight`’s alarm/bedtime helpers and give you simple service calls for UI bindings/blueprints. ([GitHub][6])

### 2.2 Alarm plumbing (`alarm.py`) – core async handlers

```python
# custom_components/eight_sleep_plus/alarm.py
from __future__ import annotations
from datetime import datetime
from homeassistant.core import HomeAssistant, ServiceCall
from .coordinator import EightDeviceCoordinator
from .const import DOMAIN

WEEKDAYS = {"monday","tuesday","wednesday","thursday","friday","saturday","sunday"}

async def async_set_recurring_alarm(hass: HomeAssistant, call: ServiceCall) -> None:
    side = call.data["side"]
    time_str = call.data["time"]
    weekdays = {d: (d in call.data.get("weekdays", [])) for d in WEEKDAYS}
    vpower = call.data.get("vibration_power")
    thermal = call.data.get("thermal_wake", True)
    light_sleep = call.data.get("smart_light_sleep", True)

    coordinator: EightDeviceCoordinator = hass.data[DOMAIN]["coordinator"]
    user = coordinator.get_user_for_side(side)
    await user.update_alarm_data()
    alarm = user.get_alarm_by_time(time_str)
    if alarm:
        await user.set_alarm_direct(
            alarm_id=alarm["id"], enabled=True, weekdays=weekdays,
            vibration_power=vpower, thermal_enabled=thermal, smart_light_sleep=light_sleep
        )
    else:
        # create then configure
        await user.create_alarm(time_str)
        await user.update_alarm_data()
        alarm = user.get_alarm_by_time(time_str)
        if alarm:
            await user.set_alarm_direct(
                alarm_id=alarm["id"], enabled=True, weekdays=weekdays,
                vibration_power=vpower, thermal_enabled=thermal, smart_light_sleep=light_sleep
            )
    await coordinator.async_request_refresh()

async def async_set_one_off_alarm(hass: HomeAssistant, call: ServiceCall) -> None:
    side = call.data["side"]
    dt: datetime = call.data["datetime"]
    vpower = call.data.get("vibration_power")
    thermal = call.data.get("thermal_wake", True)
    light_sleep = call.data.get("smart_light_sleep", True)

    coordinator: EightDeviceCoordinator = hass.data[DOMAIN]["coordinator"]
    user = coordinator.get_user_for_side(side)
    await user.set_one_off_alarm(dt, vibration_power=vpower, thermal_enabled=thermal, smart_light_sleep=light_sleep)
    await coordinator.async_request_refresh()

async def async_edit_bedtime(hass: HomeAssistant, call: ServiceCall) -> None:
    side = call.data["side"]
    bedtime = call.data["bedtime"]
    b = call.data.get("bedtime_level")
    i = call.data.get("initial_level")
    f = call.data.get("final_level")
    coordinator: EightDeviceCoordinator = hass.data[DOMAIN]["coordinator"]
    user = coordinator.get_user_for_side(side)
    await user.set_bedtime_schedule(bedtime, bedtime_temp=b, initial_sleep_temp=i, final_sleep_temp=f)
    await coordinator.async_request_refresh()
```

These map 1:1 to `pyEight` methods documented in the repo (and consistent with what merged in 1.0.22 betas). ([GitHub][2])

### 2.3 Climate presentation (degrees display)

Eight Sleep’s API uses a **unit‑less -100..100 “level”**; the README still lists a TODO to **translate to degrees** for ease of use. Eight Sleep advertises 55–110 °F water/air temperature range; so a linear, user‑friendly display is:

```
F(level) = 82.5 + 0.275 * level   (≈ 55°F @ -100, ≈110°F @ +100), 
C = (F − 32) * 5/9
```

Keep the device‑side command in levels to avoid drift; **only surface degrees in UI**. (This is an approximation—Eight Sleep doesn’t publish a guaranteed mapping; expose both “level” and “≈° temperature” attributes.) The climate entity (added in 1.0.21) can present **target\_temperature** in °C/°F while the setter forwards a level value under the hood. ([GitHub][2], [Eight Sleep][7])

### 2.4 Events & device actions

Add HA events for better automations:

* `eight_sleep_plus.alarm_started` (side, alarm\_id, time)
* `eight_sleep_plus.priming_done` (side)
* `eight_sleep_plus.snore_mitigation_active` (bool → when toggles)
* `eight_sleep_plus.autopilot_adjust` (side, new\_level, reason)

And **device actions** (HA Device Automation) for the bed device: “Prime”, “Turn side on/off”, “Snooze alarm”, “Stop alarm”, “Dismiss upcoming”, “Set base preset”, “Nudge head/feet”.

---

## 3) Autopilot+ (local loop + LLM policy)

**Design goals**

* Real‑time loop every \~5 min using **local heuristics** (no cloud dependency).
* Optional **LLM** summaries to tune nightly policy (not in the tight loop).
* Blend bed data (HR, RR, room temp, bed state/levels) with **Apple Watch** signals.

> Integration cadence: bed side sensors **update \~5 minutes**; base controls update **\~60s**. Presence is retroactive; we’ll avoid relying solely on it for “out of bed.” ([GitHub][1])

### 3.1 Signals we’ll use

* From Eight Sleep: HR, HRV, RR, bed/target temp, bed state type (`off | smart:bedtime | smart:initial | smart:final`), room temperature, presence start/end, snore mitigation (if base), head/feet angle. ([GitHub][1])
* From Apple Watch (two options):

  * **iOS Companion App**: steps, distance, activity state, location tracker (no direct Health/HR). Useful for “I’m up and moving.” ([companion.home-assistant.io][8])
  * **Health Auto Export** (or Shortcut webhook): push heart‑rate, wrist‑temp deviation, sleep/wake events to HA via webhook helpers. This is community‑standard when you need Apple Health specifics in HA. ([Home Assistant Community][9], [Reddit][10])

### 3.2 Control strategy (practical & explainable)

* **In‑loop controller (every 5 min while in bed)**:
  Simple **proportional rules** around a per‑user “comfort baseline” learned from recent nights. Example logic:

  * If **RR > (rolling\_median + 2)** or **HR rising > threshold** and skin‑proxy (target–room) suggests warm, **cool by −2 to −5 levels.**
  * If **HRV dips** and we’re in `smart:initial` stage, nudge **cooler**; in `smart:final`, nudge **warmer** to reduce early wake.
  * If **snore mitigation active** or RR surges, **raise head angle** +5° up to a max; drop back when recovered.
  * Rate limit: **no more than ±5 levels / 10 min**; never adjust in final 20 min before an active alarm unless temperature is far off baseline.

* **Nightly LLM policy tuner (optional)**:
  After each night, pass a **redacted summary** (no timestamps or PII; aggregates only) to an LLM and let it adjust **stage target levels**, **angle ceilings**, and **priming temperature** for the next night. Keep decisions explainable and bounded (min/max).

**LLM microservice (FastAPI; runs locally in Docker)**

```python
# services/autopilot/main.py  (runs on your Pi 5)
from fastapi import FastAPI
from pydantic import BaseModel
from typing import Literal

class Sample(BaseModel):
    # 60–120 min aggregates from last night (per stage)
    stage: Literal["bedtime","initial","final"]
    avg_hr: float; avg_rr: float; avg_hrv: float
    room_c: float; bed_level_avg: float
    wakes: int; cold_events: int; hot_events: int

class Request(BaseModel):
    side: Literal["left","right"]
    profile: Literal["conservative","balanced","aggressive"]
    samples: list[Sample]

class Decision(BaseModel):
    bedtime_level: int; initial_level: int; final_level: int
    head_angle_max: int; foot_angle_max: int
    notes: str

app = FastAPI()

@app.post("/decide", response_model=Decision)
def decide(req: Request):
    # bounded, explainable adjustments (no model call if you want it fully local)
    # (If you want LLM: call OpenAI here with a system prompt, pass only aggregates.)
    delta = {"conservative":1,"balanced":2,"aggressive":3}[req.profile]
    hot = sum(1 for s in req.samples if s.hot_events> s.cold_events)
    cold = sum(1 for s in req.samples if s.cold_events> s.hot_events)
    # naive policy: bias cooler if more hot events than cold events
    bias = (-delta if hot>cold else delta if cold>hot else 0)
    return Decision(
        bedtime_level=int(round(req.samples[0].bed_level_avg + bias)),
        initial_level=int(round(req.samples[1].bed_level_avg + bias)),
        final_level=int(round(req.samples[2].bed_level_avg + (bias//2))),
        head_angle_max=20 + (5 if hot>cold else 0),
        foot_angle_max=10,
        notes="Rule-based adjustment; bias=%d" % bias
    )
```

If you **do** want an LLM call, insert an OpenAI chat completion in `/decide` with your key, using *only aggregate metrics* (privacy‑first). The in‑loop controller remains local.

> Why this split? The 5‑minute cadence and need for bounded, safe actions make deterministic rules ideal; an LLM can **summarize trends** and **retune tomorrow’s targets**. Also avoids sending raw sleep streams off‑box. ([trufflesecurity.com][5])

---

## 4) Lovelace dashboard (Eight Sleep “app‑like”)

Uses standard + popular custom cards (via HACS): **Mushroom**, **button‑card**, **apexcharts‑card**, **slider‑entity-row**, **stack‑in‑card**, **layout‑card**. The layout mirrors Eight Sleep: per‑side controls at top, analytics in the middle, alarm editor & base controls below.

> The integration already exposes **head/feet angle numbers**, **snore mitigation**, **base presets**, and **next alarm** + alarm switches. We bind those here. ([GitHub][1])

```yaml
# dashboards/eight_sleep/view.yaml
title: Eight Sleep
type: sections
sections:
  - type: grid
    cards:
      # --- Luca (Left) ---
      - type: vertical-stack
        cards:
          - type: custom:stack-in-card
            cards:
              - type: custom:mushroom-title-card
                title: Luca
                subtitle: Pod status: {{ states('sensor.luca_eight_sleep_side_bed_state_type') }}
              - type: entities
                entities:
                  - entity: climate.luca_eight_sleep
                    name: Temperature
                    secondary_info: "Target: {{ state_attr('climate.luca_eight_sleep','temperature') }}°  (≈{{ state_attr('climate.luca_eight_sleep','eight_estimated_temp') }})"
                    show_state: true
                    type: custom:slider-entity-row
                  - entity: switch.luca_next_alarm   # from integration
                    name: Next Alarm Enabled
                  - entity: sensor.luca_eight_sleep_side_next_alarm
                    name: Next Alarm Time
              - type: custom:button-card
                name: Prime Now
                tap_action: { action: call-service, service: eight_sleep_plus.prime_pod, service_data: { entity_id: sensor.luca_eight_sleep_side_heart_rate } }
              - type: conditional
                conditions: [{ entity: binary_sensor.eight_sleep_snore_mitigation, state: "on"}]
                card:
                  type: markdown
                  content: "Snore mitigation active — raising head."
          - type: entities
            title: Smart Alarm (Recurring)
            entities:
              - input_datetime.luca_alarm_time
              - type: custom:button-card
                name: Save Alarm
                tap_action:
                  action: call-service
                  service: eight_sleep_plus.set_smart_alarm
                  service_data:
                    side: left
                    time: '[[[ return states["input_datetime.luca_alarm_time"].state ]]]'
                    weekdays: ["monday","tuesday","wednesday","thursday","friday"]
          - type: custom:apexcharts-card
            header:
              title: Night Analytics (Luca)
            series:
              - entity: sensor.luca_eight_sleep_side_heart_rate
              - entity: sensor.luca_eight_sleep_side_breath_rate
              - entity: sensor.luca_eight_sleep_side_hrv
            graph_span: 24h

      # --- Partner (Right) --- (duplicate structure, with right-side entities)
      - type: vertical-stack
        cards:
          # ... mirror Luca’s stack using partner entities ...
```

> If you prefer an all‑in‑one custom card (TypeScript/Lit), we can bundle a `eight-sleep-card` later; this YAML gets us 95% of the app feel now.

---

## 5) Automations & blueprints

### 5.1 **Smart Priming** (per‑side, presence‑aware)

**Inputs**: per‑side temperature entity or climate, person entities, device trackers (iOS), optional Apple Watch step sensor (webhook), Hue group, Lutron shades, Apple TV/Projector, Ecobee thermostat.

```yaml
# blueprints/automation/luca/eight_sleep_smart_priming.yaml
blueprint:
  name: Eight Sleep – Smart Priming (per side)
  domain: automation
  input:
    side: { name: Side, selector: { select: { options: ["left","right"] } } }
    person: { name: Person, selector: { entity: { domain: person } } }
    device_tracker: { name: Device tracker, selector: { entity: { domain: device_tracker } } }
    steps_sensor: { name: Steps sensor (optional), default: '', selector: { entity: {} } }
    prime_after: { name: Not before time, default: '20:30:00', selector: { time: {} } }
    prime_lead_min: { name: Lead minutes before typical bedtime, default: 30, selector: { number: { min: 0, max: 120 } } }
    typical_bedtime: { name: Typical bedtime helper, selector: { entity: { domain: input_datetime } } }
    prime_service_target: { name: Any Eight Sleep side entity (for service target), selector: { entity: { domain: sensor } } }
trigger:
  - platform: time
    at: !input prime_after
  - platform: state
    entity_id: !input device_tracker
    to: 'home'
  - platform: state
    entity_id: !input steps_sensor
condition:
  - condition: state
    entity_id: !input device_tracker
    state: 'home'
  - condition: template
    value_template: >
      {% set lead = (states('input_number.prime_lead_min')|int(30)) %}
      {% set tb = states(!input typical_bedtime) %}
      {% if tb in ['unknown','unavailable'] %} true
      {% else %}
      {{ now().strftime('%H:%M') >= (strptime(tb, '%H:%M:%S') - timedelta(minutes=lead)).strftime('%H:%M') }}
      {% endif %}
action:
  - service: eight_sleep_plus.prime_pod
    target: { entity_id: !input prime_service_target }
  - choose:
      - conditions:
          - condition: template
            value_template: "{{ ! (now().hour < 18) }}"
        sequence:
          - service: light.turn_on
            target: { entity_id: light.bedroom }
            data: { brightness_pct: 15, color_temp: 450 }
          - service: cover.close_cover
            target: { entity_id: cover.bedroom_shades }
          - service: climate.set_temperature
            target: { entity_id: climate.ecobee_home }
            data: { temperature: 69 }
mode: restart
```

### 5.2 **Autopilot+ loop** (5‑min cadence while in bed)

```yaml
# blueprints/automation/luca/eight_sleep_autopilot_loop.yaml
blueprint:
  name: Eight Sleep – Autopilot+ Loop
  domain: automation
  input:
    side_entities:
      name: Side package
      description: HR, RR, HRV, bed/target temp, room temp, base angle numbers, etc. (use a group or provide individually)
    autopilot_mode: { name: Autopilot select, selector: { entity: { domain: select } } }
    decide_url: { name: Local policy endpoint, default: 'http://127.0.0.1:8000/decide', selector: { text: {} } }
trigger:
  - platform: time_pattern
    minutes: "/5"
condition:
  - condition: state
    entity_id: binary_sensor.bed_presence    # prefer your own occupancy signal
    state: "on"
action:
  - variables:
      profile: "{{ states(!input autopilot_mode) or 'balanced' }}"
  - service: rest_command.autopilot_decide         # define a REST command to your FastAPI
    data:
      payload: >
        {{ {
          "side":"left",
          "profile": profile,
          "samples":[
            {"stage":"bedtime","avg_hr":states('sensor.hr')|float, "avg_rr":states('sensor.rr')|float, "avg_hrv":states('sensor.hrv')|float,
             "room_c": states('sensor.room_temp')|float, "bed_level_avg": state_attr('climate.luca_eight_sleep','eight_level')|int,
             "wakes": states('sensor.wakes')|int(0), "cold_events": states('sensor.cold_events')|int(0), "hot_events": states('sensor.hot_events')|int(0)},
            # initial & final segments omitted for brevity—same shape
          ]
        } | tojson }}
  - choose:
      - conditions: []   # after REST returns, parse `bedtime/initial/final` targets and set via service
        sequence:
          - service: climate.set_temperature
            target: { entity_id: climate.luca_eight_sleep }
            data: { temperature: "{{ state_attr('climate.luca_eight_sleep','temperature') }}" }
mode: restart
```

*(You’ll add a `rest_command.autopilot_decide` in your HA config pointing to the FastAPI service; parse its response with a small script or a `python_script` to set levels/angles. I kept the YAML compact.)*

### 5.3 **Wake‑up routine** (alarm‑ or movement‑based)

```yaml
# blueprints/automation/luca/eight_sleep_wakeup_routine.yaml
blueprint:
  name: Eight Sleep – Wake Up Routine
  domain: automation
  input:
    alarm_switch: { selector: { entity: { domain: switch } } }
    presence_sensor: { selector: { entity: { domain: binary_sensor } } }
    steps_sensor: { default: '', selector: { entity: {} } }
    lights: { selector: { entity: { domain: light } } }
    shades: { selector: { entity: { domain: cover } } }
    media_player: { selector: { entity: { domain: media_player } } }
trigger:
  - platform: state
    entity_id: !input alarm_switch
    to: 'on'
  - platform: state
    entity_id: !input presence_sensor
    to: 'off'
    for: '00:05:00'
  - platform: numeric_state
    entity_id: !input steps_sensor
    above: 50
condition:
  - condition: time
    after: '05:00:00'
    before: '11:30:00'
action:
  - service: light.turn_on
    target: { entity_id: !input lights }
    data: { brightness_pct: 35, transition: 5 }
  - service: cover.open_cover
    target: { entity_id: !input shades }
  - service: media_player.play_media
    target: { entity_id: !input media_player }
    data: { media_content_id: "spotify:playlist:37i9dQZF1DXc8kgYqQLMfH", media_content_type: "music" }
  - service: eight_sleep_plus.side_off
    data: { entity_id: sensor.luca_eight_sleep_side_heart_rate }
mode: restart
```

**Behavior notes**

* **Per‑side** independence: if you leave and your partner stays, the routine only acts on your side (check `person`/`device_tracker` per blueprint instance).
* “Out of bed” uses **Apple Watch steps** or **presence off** for 5 minutes (presence from Eight Sleep is delayed; steps/location is faster). ([GitHub][1], [companion.home-assistant.io][8])

---

## 6) Smart Alarm editor (works now)

With the merged PRs & `pyEight`, the integration supports **editing/creating alarms** (recurring + one‑off). If your current installation can only toggle on/off, update to **1.0.21+** and, ideally, to the **1.0.22 beta** which merged alarm updates. Then bind the UI buttons/time pickers to the `set_smart_alarm` and `set_one_off_alarm` services we added. ([GitHub][2])

> Release notes: “Adding alarm switches”, “Next alarm switch”, “Add routine editing”, “Add one off alarms”. ([GitHub][2])

Example HA service call (from Dev Tools → Services):

```yaml
service: eight_sleep_plus.set_smart_alarm
data:
  side: left
  time: "06:45"
  weekdays: ["monday","tuesday","wednesday","thursday","friday"]
  vibration_power: 70
  thermal_wake: true
  smart_light_sleep: true
target:
  entity_id: sensor.luca_eight_sleep_side_heart_rate
```

---

## 7) Apple Watch data path (practical options)

* **Built‑in** (no extra apps): iOS Companion exposes **steps/activity/location**; this is enough to detect “awake/out of bed” and presence for automations. It **does not** expose Health HR/sleep/temperature streams. ([companion.home-assistant.io][8])
* **Health Auto Export** (recommended): schedule exports or Shortcuts to **webhook** HR/temperature/sleep events to HA template sensors. Community‑tested path when you need HR/skin‑temp from Apple Watch in HA. ([Home Assistant Community][9], [Reddit][10])

---

## 8) Security & resilience

* Add a **local kill‑switch** (`input_boolean.eight_sleep_local_lockout`). When ON, the integration refuses to call the Eight Sleep cloud and your Autopilot loop also halts updates.
* Rate‑limit all API writes (cooldown between level changes, base angle nudges).
* On HTTP 401/429, **backoff** and avoid flapping.
* Keep LLM calls **local** or strictly **aggregate** metrics. Given public research about questionable remote access to Pods, **assume cloud may be unavailable**; your routines and autopilot should remain useful offline. ([trufflesecurity.com][5])

---

## 9) What you’ll see after this fork

* **Entities** (per side)

  * `climate.<side>_eight_sleep` – target temp in °C/°F (≈presented), attribute `eight_level` for the underlying API level.
  * `sensor.<side>_eight_sleep_side_heart_rate`, `..._breath_rate`, `..._hrv`, `..._sleep_scores`, `..._next_alarm`, `..._bed_state_type`, `..._room_temperature`. ([GitHub][1])
  * Base numbers: `number.base_head_angle`, `number.base_feet_angle`, `select.base_preset`, `binary_sensor.snore_mitigation`. ([GitHub][2])
  * `select.autopilot_mode` (off/conservative/balanced/aggressive).

* **Services**

  * Existing: heat\_set/increment, side\_on/off, prime\_pod, alarm\_\* (snooze/stop/dismiss), start/stop\_away. ([GitHub][1])
  * New: `set_smart_alarm`, `set_one_off_alarm`, `edit_bedtime_schedule`, `autopilot_set_mode`.

* **Events**

  * `eight_sleep_plus.alarm_started`, `...priming_done`, `...snore_mitigation_active`, `...autopilot_adjust`.

---

## 10) Install/upgrade checklist

1. **Fork & clone** this repo as `eight_sleep_plus` (keep the domain slug stable for HACS).
2. Update **`manifest.json`** version & domain, and add `pyEight` as a dependency.
3. **HACS → Custom repositories**: add your fork, install, restart HA.
4. Upgrade to **1.0.21+** (or **1.0.22 beta**) if you’re still on an older revision to get alarm fixes. ([GitHub][2])
5. Create the HA **helpers**: `input_datetime.luca_alarm_time`, `input_datetime.partner_alarm_time`, `input_boolean.eight_sleep_local_lockout`, `input_select.autopilot_mode`.
6. Import the **blueprints** & instantiate per side.
7. Add the **Lovelace view** or create a new Dashboard and paste the YAML from §4.
8. (Optional) Deploy the **FastAPI microservice** (`uvicorn services.autopilot.main:app --host 0.0.0.0 --port 8000`) in Docker on the Pi 5; add `rest_command.autopilot_decide` to HA pointing to it.

---

## 11) Smart alarm UX parity

* The Lovelace view includes **time pickers** and “Save Alarm” per side; behind the button we call the **recurring alarm** service.
* One‑off alarms use a datetime picker + `set_one_off_alarm`.
* The integration’s **“next alarm” switch** lets you skip only the next instance without disabling the series (as added in 1.0.20+). ([GitHub][2])

---

## 12) Caveats & assumptions (explicit)

* **Degrees mapping** is UI‑approximate; Eight Sleep doesn’t guarantee any linear relation between level and actual surface temp. We surface both the **level** (ground truth for API) and an **≈°F/°C** hint. ([GitHub][1], [Eight Sleep][7])
* **Presence** from Eight Sleep is delayed/retroactive; we don’t use it for wake detection—**steps/location** are primary. ([GitHub][1])
* **Apple Watch HR/sleep stages** are not available via the stock iOS Companion; use **Health Auto Export** or Shortcuts → webhook for richer signals. ([companion.home-assistant.io][8], [Home Assistant Community][9])
* **Alarm editing** requires the newer alarm endpoints; they are present in `pyEight` and referenced by the 1.0.22 beta PR. Ensure your HA install uses those bits. ([GitHub][6])

---

## 13) Why this hits your goals

* **UI parity**: the per‑side stacks, sliders, base angles, alarm editors, and analytics mirror the official app layout while remaining native HA.
* **Autopilot “better than stock”**: bounded, explainable, **local first** controls with optional LLM policy tuning; integrates your **Apple Watch** to start/stop/adjust intelligently.
* **Routines**: priming based on **home, time, and “nearing bedtime”**; wake‑up sequences tied to **alarm or movement**; per‑side independence.
* **Future‑proofing & safety**: device actions, events, rate limiting, backoff, and a **kill‑switch** with a bias toward **local** control. ([trufflesecurity.com][5])

---

### If you want me to push the exact fork layout as a PR-ready patch, say the word and I’ll output the repo tree with the modified files (full file contents) so you can drop them into your fork and install via HACS.

[1]: https://github.com/lukas-clarke/eight_sleep "GitHub - lukas-clarke/eight_sleep: Home Assistant Eight Sleep integration that works with Eight Sleep's new API and OAUTH2"
[2]: https://github.com/lukas-clarke/eight_sleep/releases "Releases · lukas-clarke/eight_sleep · GitHub"
[3]: https://www.home-assistant.io/blog/2023/11/01/release-202311/?utm_source=chatgpt.com "2023.11 To-do: Add release title"
[4]: https://github.com/lukas-clarke/pyEight "GitHub - lukas-clarke/pyEight: This is python code to interact with Eight Sleeps new OAuth2 API"
[5]: https://trufflesecurity.com/blog/removing-jeff-bezos-from-my-bed?utm_source=chatgpt.com "Removing Jeff Bezos From My Bed"
[6]: https://github.com/lukas-clarke/eight_sleep/pull/90 "Merge netlob alarm updates by google-labs-jules[bot] · Pull Request #90 · lukas-clarke/eight_sleep · GitHub"
[7]: https://www.eightsleep.com/?srsltid=AfmBOorKNumJqEzQLRjNsxDuxWGytdIvBaNsVzWy-P2gtPjVkkdjJVsU&utm_source=chatgpt.com "Eight Sleep | The Intelligent Bed Cooling System"
[8]: https://companion.home-assistant.io/docs/getting_started/?utm_source=chatgpt.com "Home Assistant Companion Docs: Getting Started"
[9]: https://community.home-assistant.io/t/getting-your-android-ios-fitness-data-into-ha/369895?utm_source=chatgpt.com "Getting your Android & iOS fitness data into HA"
[10]: https://www.reddit.com/r/homeassistant/comments/ukdx8p/best_way_to_get_ios_steps_in_to_ha/?utm_source=chatgpt.com "Best way to get iOS Steps in to HA : r/homeassistant"
