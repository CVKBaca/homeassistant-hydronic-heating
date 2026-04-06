# Roadmap

This document describes the architectural evolution of the hydronic heating blueprints.

---

## Current Architecture (v3.0) — Released 2026-04-07

The current system uses **4 blueprints**:

| Blueprint | Instances | Purpose |
|-----------|-----------|---------|
| `shelly_trv_controller` | 1× per TRV | Valve control + temperature sync + target temperature |
| `hydronic_boiler_controller` | 1× total | Boiler on/off with short-cycling protection |
| `hydronic_heating_schedule` | 1× total | Morning/Day/Evening/Night mode switching by time + sun |
| `hydronic_presence_controller` | 1× total | Away/Holiday mode + hot water by presence |

**External prerequisites:**
- `binary_sensor.thermostat_*` — 1× per room
- `sensor.heating_status` — 1× (heating season sensor)
- `input_number.*_current` — 1× per room
- `input_number.*_morning/day/evening/night/away_mode` — 5× per room
- `input_number.heating_temperature_house_holiday_mode` — 1× global
- `input_select.heating_mode` — 1× (Morning/Day/Evening/Night/Away/Holiday)
- `heating_apply_mode` plain automation — 1× (maps mode to room temperatures)

For an 11-TRV installation: **11 + 3 = 14 automation instances** + 1 plain automation (down from 34 in v1.x).

---

## Previous Architecture (v2.0) — Released 2026-04-05

The current system uses **2 blueprints** (down from 4 in v1.x):

| Blueprint | Instances | Purpose |
|-----------|-----------|---------|
| `shelly_trv_controller` | 1× per TRV | Valve control + temperature sync + target temperature |
| `hydronic_boiler_controller` | 1× total | Boiler on/off with protection |

**External prerequisites:**
- `binary_sensor.thermostat_*` — 1× per room
- `sensor.heating_status` — 1× (heating season sensor)
- `input_number.*_current` — 1× per room

For an 11-TRV installation: **11 + 1 = 12 automation instances** (down from 34 in v1.x).

> **`sensor.valve_position_*` is no longer required.** The proportional lookup table is now computed inside `shelly_trv_controller`.

---

## v1.x Architecture (archived)

The previous system used **4 blueprints** (reduced from 5 in v1.0 by merging the boiler turn on/off into a single controller):

| Blueprint | Instances | Purpose |
|-----------|-----------|---------|
| `shelly_trv_set_valve_position` | 1× per TRV | Proportional valve control |
| `shelly_trv_sync_temperature` | 1× per TRV | External temperature to TRV |
| `shelly_trv_set_target_temperature` | 1× per TRV | Target temp from input_number |
| `hydronic_boiler_controller` | 1× total | Boiler on/off with protection |

Additional prerequisite: `sensor.valve_position_*` — 1× per TRV (proportional lookup table, UI template sensor).

For a 10-TRV installation: **4 × 10 + 1 = 31 automation instances** + 10 external helper sensors.

The v1.x blueprint files are still included in this repository for users who have not yet migrated.

---

## v2.1 — Configurable Lookup Table (planned)

**Goal:** Allow users to adjust the delta→position proportional curve without modifying YAML.

Currently the lookup table is hardcoded inside `shelly_trv_controller`. A future version could expose it as a blueprint input (e.g. a list of threshold/position pairs), or offer preset curves (aggressive / balanced / gentle).

This matters for users with very large radiators (small delta → 100% open) or very small radiators (need steeper curve).

---

## v2.2 — Generic TRV Support (planned)

**Goal:** Remove the dependency on Shelly TRV MQTT topics.

Currently all MQTT topics are constructed from `mqtt_topic_base`. A more generic version would:
- Support other TRV brands using `climate` entity control (HA native)
- Keep the Shelly MQTT path as an optional "fast path" for lower latency

This would allow the blueprints to work with any TRV that exposes a `climate` entity, not just Shelly.

---

## Design Notes

### Why 2 blueprints instead of 1?

The split between TRV controller and boiler controller is intentional:
- **Different trigger logic** — TRV controller reacts to temperature changes; boiler controller reacts to aggregate demand
- **Different instances** — TRV controller runs 1× per TRV; boiler controller runs once for the whole system
- **Independent failure domains** — a bug in TRV logic does not affect boiler safety

### Why `mode: restart` in the Boiler Controller

The boiler Turn ON action contains a `wait_template` (short-cycling protection) that can block for up to `min_cycle_protection` minutes. If all thermostats turn off during this wait, `mode: restart` cancels the wait and re-evaluates — preventing the boiler from turning on after demand has disappeared.

With `mode: single`, the Turn OFF path would be blocked during the wait. `mode: restart` is the correct choice here.

### Why proportional control instead of on/off?

Simple on/off TRV control (thermostat mode) causes the boiler to cycle frequently — the room reaches target temperature, TRV closes, boiler stops, room cools, TRV opens, boiler starts again. For a gas condensing boiler, frequent short cycles reduce efficiency and increase wear.

Proportional control keeps the valve partially open as the room approaches target temperature, reducing overshoot and maintaining steadier water flow through the circuit. The boiler runs for longer periods at lower output, which improves condensing efficiency.
