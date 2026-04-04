# Plan: shelly_trv_controller.yaml (v2.0)

> Created: 2026-04-04
> Status: Planning
> Author: Claude (based on v1.x analysis)

---

## Goal

Merge 3 per-TRV blueprints into 1:
- `shelly_trv_set_valve_position`
- `shelly_trv_sync_temperature`
- `shelly_trv_set_target_temperature`

**Result:** 33 automation instances → 11, one per TRV.

---

## Key Changes vs v1.x

| Area | v1.x | v2.0 |
|------|------|------|
| Blueprints per TRV | 3 | 1 |
| Automation instances (11 TRVs) | 33 | 11 |
| `sensor.valve_position_*` | Required (UI helper) | Eliminated — computed inline |
| Delta source | `trv_entity.temperature` | `target_temp_input` (more reliable) |
| MQTT topics | 3 separate inputs | 1 `mqtt_topic_base`, suffixes derived |
| Retry loop | Infinite | Max 5 attempts |
| `hysteresis_threshold` | Hardcoded 5% | Parameter (0–5, default 5) |
| mode | single | queued, max: 2 |

### Why `target_temp_input` instead of `trv_entity.temperature` for delta

`trv_entity.temperature` is updated only after the TRV sends back its state via MQTT.
If we publish `target_t` and immediately compute valve position, the climate entity may not
have updated yet — causing a stale delta. Using `target_temp_input` directly is the ground
truth and avoids this timing dependency.

### Why `valve_position_sensor` is eliminated

The sensor existed to make desired valve position visible and graphable in the HA UI.
In practice, the same diagnostic information is available via automation trace.
Eliminating it removes 11 UI helpers that users must create manually.

---

## Inputs

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `trv_entity` | ✅ | — | Climate entity of the Shelly TRV |
| `aqara_sensor` | ✅ | — | External room temperature sensor |
| `trv_temperature_sensor` | ✅ | — | TRV internal temp sensor (for stabilization check) |
| `current_valve_entity` | ✅ | — | `number.shelly_trv_*_valve_position` |
| `target_temp_input` | ✅ | — | `input_number.*_current` |
| `mqtt_topic_base` | ✅ | — | e.g. `shellies/shellytrv_bedroom_1/thermostat/0/command` |
| `window_sensor` | ⬜ | `sun.sun` | Window/door sensor. `sun.sun` is never `on` — safe no-op default. |
| `frost_temp` | ⬜ | `8` | Temperature (°C) when window is open |
| `heating_status_sensor` | ⬜ | `sensor.heating_status` | Returns `On`/`Off` |
| `boiler_sensor` | ⬜ | `binary_sensor.protherm_24` | Binary sensor for short-cycling protection |
| `hysteresis_threshold` | ⬜ | `5` | Min valve position change (%) to trigger MQTT send |

---

## Variables

```yaml
variables:
  trv_entity: !input trv_entity
  aqara_sensor: !input aqara_sensor
  trv_temperature_sensor: !input trv_temperature_sensor
  current_valve_entity: !input current_valve_entity
  target_temp_input: !input target_temp_input
  mqtt_topic_base: !input mqtt_topic_base
  window_sensor: !input window_sensor
  frost_temp: !input frost_temp
  heating_status_sensor: !input heating_status_sensor
  boiler_sensor: !input boiler_sensor
  hysteresis_threshold: !input hysteresis_threshold

  # Derived
  mqtt_valve_pos: "{{ mqtt_topic_base }}/valve_pos"
  mqtt_ext_t: "{{ mqtt_topic_base }}/ext_t"
  mqtt_target_t: "{{ mqtt_topic_base }}/target_t"

  # Computed once, reused in condition and retry loop
  desired_position: >
    {% set delta = states(target_temp_input) | float(0) - states(aqara_sensor) | float(25) %}
    {% if states(heating_status_sensor) != 'On' %}0
    {% elif delta <= -0.10 %}0
    {% elif delta <= -0.05 %}13
    {% elif delta <= 0.00 %}18
    {% elif delta <= 0.05 %}25
    {% elif delta <= 0.10 %}33
    {% elif delta <= 0.15 %}40
    {% elif delta <= 0.20 %}48
    {% elif delta <= 0.25 %}55
    {% elif delta <= 0.30 %}63
    {% elif delta <= 0.35 %}70
    {% elif delta <= 0.40 %}78
    {% elif delta <= 0.45 %}85
    {% elif delta <= 0.50 %}93
    {% else %}100
    {% endif %}
```

---

## Triggers

```yaml
triggers:
  # valve_pos + sync_temp
  - trigger: state
    entity_id: !input aqara_sensor

  # target_t + valve_pos
  - trigger: state
    entity_id: !input target_temp_input

  # frost protection
  - trigger: state
    entity_id: !input window_sensor
    to: 'on'
    for:
      seconds: 15

  # restore after window closes
  - trigger: state
    entity_id: !input window_sensor
    to: 'off'
    for:
      minutes: 5

  # safety net — catches any missed triggers
  - trigger: time_pattern
    minutes: /15
```

---

## Actions

Three sequential blocks. Each evaluates its own condition — only relevant blocks execute.

### Block 1 — Set target temperature

**When to run:** `target_temp_input` changed, `window_sensor` changed, or `time_pattern` safety net.

```
if window is open:
  if trv target temp != frost_temp:
    publish frost_temp to mqtt_target_t
else:
  if trv target temp != target_temp_input:
    publish target_temp_input to mqtt_target_t
```

### Block 2 — Sync external temperature

**When to run:** `aqara_sensor` changed or `time_pattern` safety net.
**Condition:** `heating_status == On` AND `current_valve > 0`

```
if aqara changed recently (< 5 min) AND |aqara - trv_internal| > 0.04°C:
  wait for TRV internal sensor to stabilize (up to 5 min)
publish aqara value to mqtt_ext_t
```

### Block 3 — Set valve position

**When to run:** Any trigger.
**Condition:** `|desired_position - actual| >= hysteresis_threshold` OR `(heating Off AND actual > 0)`

```
if boiler changed state recently (< min_cycle_protection):
  wait until min_cycle_protection elapsed (timeout: min_cycle_protection * 60 + 60s)

repeat (max 5 attempts):
  publish desired_position to mqtt_valve_pos
  delay 5s
until: actual == desired OR repeat.index >= 5
```

If after 5 attempts actual != desired: the /15 safety net will retry.

---

## Mode

```yaml
mode: queued
max: 2
```

**Why queued:** If a new trigger arrives while Block 3 retry loop is running,
`mode: single` would discard it. `queued` holds one pending execution — handles
rapid temperature changes without losing updates. `max: 2` prevents queue buildup
if TRV is offline for extended period.

---

## Migration from v1.x (existing installation)

1. Create `shelly_trv_controller.yaml` blueprint
2. Upload to HA server, reload blueprints
3. Create 11 new automation instances (one per TRV)
4. Test for 2–3 days alongside existing v1.x automations (disable, not delete)
5. If stable: delete old 33 automations + old 3 blueprints
6. Delete `sensor.valve_position_*` UI helpers (11×) — no longer needed

**Risk:** Low — v1.x automations remain active as fallback during testing.

---

## Future (v2.1)

Configurable lookup table — expose delta thresholds and positions as blueprint inputs.
Relevant for installations with different radiator sizing or flow rates.
Only implement if there is actual user demand.
