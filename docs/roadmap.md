# Roadmap

This document describes planned improvements and architectural evolution of the hydronic heating blueprints.

---

## Current Architecture (v1.x)

The current system uses **4 blueprints** (reduced from 5 in v1.0 by merging the boiler turn on/off into a single controller):

| Blueprint | Instances | Purpose |
|-----------|-----------|---------|
| `shelly_trv_set_valve_position` | 1× per TRV | Proportional valve control |
| `shelly_trv_sync_temperature` | 1× per TRV | External temperature to TRV |
| `shelly_trv_set_target_temperature` | 1× per TRV | Target temp from input_number |
| `hydronic_boiler_controller` | 1× total | Boiler on/off with protection |

**External prerequisites** (created manually, not in blueprints):
- `sensor.valve_position_*` — 1× per TRV (proportional lookup table)
- `binary_sensor.thermostat_*` — 1× per room
- `sensor.heating_status` — 1× (heating season sensor)
- `input_number.*_current` — 1× per room

For a 10-TRV installation: **4 blueprints × 10 instances + 1 boiler = 31 automation instances** plus **10 external helper sensors**.

---

## v2.0 — Consolidated TRV Controller (planned)

**Goal:** Reduce 3 per-TRV blueprints to 1, and eliminate `sensor.valve_position_*` external helpers.

### Proposed: `shelly_trv_controller.yaml`

A single blueprint replacing `set_valve_position` + `sync_temperature` + `set_target_temperature`.

**Key change:** The valve position lookup table (currently in `sensor.valve_position_*`) moves inside the blueprint. The blueprint computes the desired position internally from `trv_entity` and `aqara_sensor` inputs.

**Inputs:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `trv_entity` | ✅ | Climate entity of the Shelly TRV |
| `aqara_sensor` | ✅ | External room temperature sensor |
| `current_valve_entity` | ✅ | `number.shelly_trv_*_valve_position` |
| `target_temp_input` | ✅ | `input_number` with desired room temperature |
| `mqtt_topic_base` | ✅ | Base MQTT topic (e.g. `shellies/shellytrv_bedroom_1/thermostat/0/command`) |
| `window_sensor` | ⬜ | Window/door sensor for frost protection |
| `frost_temp` | ⬜ | Frost temperature when window is open (default: 8°C) |
| `heating_status_sensor` | ⬜ | Heating season sensor (default: `sensor.heating_status`) |
| `boiler_sensor` | ⬜ | Boiler sensor for short-cycling protection |

**What this eliminates:**
- `sensor.valve_position_*` — computed internally, no UI helper needed per TRV
- 2 out of 3 per-TRV automation instances

**Trade-off:** The valve position is no longer visible as a named sensor in the HA UI. You lose the ability to monitor "desired vs actual" position without opening the automation trace. If observability matters to you, keep the external sensor approach (v1.x).

**Result for 10-TRV installation:** 10 automation instances + 1 boiler (down from 31).

---

## v2.1 — Configurable Lookup Table (planned)

**Goal:** Allow users to adjust the delta→position proportional curve without modifying YAML.

Currently the lookup table is hardcoded. A future version could expose it as a blueprint input (e.g. a list of threshold/position pairs), or offer preset curves (aggressive / balanced / gentle).

This matters for users with very large radiators (small delta → 100% open) or very small radiators (need steeper curve).

---

## v2.2 — Generic TRV Support (planned)

**Goal:** Remove the dependency on Shelly TRV MQTT topics.

Currently all three MQTT topics are constructed explicitly. A more generic version would:
- Support other TRV brands using `climate` entity control (HA native)
- Keep the Shelly MQTT path as an optional "fast path" for lower latency

This would allow the blueprints to work with any TRV that exposes a `climate` entity, not just Shelly.

---

## Design Notes

### Why 4 blueprints instead of 2?

In retrospect, the ideal architecture would be exactly 2 blueprints:
1. `shelly_trv_controller` — all TRV logic (valve + temperature sync + target)
2. `hydronic_boiler_controller` — boiler logic (already there)

The current split into 3 TRV blueprints exists for two reasons:
1. **Incremental development** — blueprints were extracted from existing automations one concern at a time
2. **Observability** — keeping `sensor.valve_position_*` as external entities makes it easy to monitor and graph the proportional control in the HA UI

v2.0 will address this.

### Why `sensor.valve_position_*` is external (v1.x)

The proportional lookup table lives outside the blueprint as a UI helper sensor because:
- It makes the computed position visible and graphable in HA dashboards
- It can be read by other automations (e.g. a dashboard card showing all valve positions)
- It decouples the position computation from the MQTT send logic

The downside is that it creates 10+ additional entities that users must configure manually.

### Why `mode: restart` in the Boiler Controller

The boiler Turn ON action contains a `wait_template` (short-cycling protection) that can block for up to `min_cycle_protection` minutes. If all thermostats turn off during this wait, `mode: restart` cancels the wait and re-evaluates — preventing the boiler from turning on after demand has disappeared.

With `mode: single`, the Turn OFF path would be blocked during the wait. `mode: restart` is the correct choice here.
