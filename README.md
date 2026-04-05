# Hydronic Heating for Home Assistant

A set of Home Assistant blueprints for controlling hydronic (hot water) heating systems with **Shelly TRV** thermostatic radiator valves and a gas/oil boiler.

The system implements **proportional valve control** — instead of simple on/off, each radiator valve opens proportionally to the temperature deficit. The boiler is controlled automatically based on actual heating demand.

---

## ⚠️ Important Safety Requirement

**At least one radiator in the hydraulic circuit must remain permanently open** (manually set, not controlled by this system).

If all radiators close simultaneously, the boiler and circulation pump will push water into a closed circuit — this can damage or destroy your equipment.

---

## System Architecture (v2.0)

```
[Aqara temp sensor] ──►
[climate target temp] ──►  [shelly_trv_controller] ──► [MQTT → Shelly TRV valve_pos]
[input_number target] ──►                            ──► [MQTT → Shelly TRV ext_t]
[window sensor]       ──►                            ──► [MQTT → Shelly TRV target_t]
                                │
                     [number.shelly_trv_*_valve_position]
                                │
                     [binary_sensor.thermostat_*] (>18%)
                                │
              ┌─────────────────┘
              │
   [hydronic_boiler_controller] ──► [boiler switch]
```

One `shelly_trv_controller` instance per TRV. One `hydronic_boiler_controller` total.

For an 11-TRV installation: **11 + 1 = 12 automation instances** (previously 34).

---

## Prerequisites

Before installing the blueprints, create the following entities manually. See [docs/prerequisites.md](docs/prerequisites.md) for detailed instructions.

| Entity | Type | Description |
|--------|------|-------------|
| `binary_sensor.thermostat_*` | YAML Template Sensor | `true` when valve position > 18% (room needs heat) |
| `sensor.heating_status` | UI Template Sensor | Returns `On`/`Off` based on outdoor temperature / season |
| `input_number.*_current` | Input Number | Target temperature per room, controllable from UI |

> **v2.0:** `sensor.valve_position_*` is **no longer required**. The valve position lookup table is now computed inside `shelly_trv_controller`.

---

## Blueprints

### 1. Shelly TRV Controller (v2.0)

**File:** `blueprints/shelly_trv_controller.yaml`

A single blueprint replacing the three separate v1.x blueprints (`set_valve_position` + `sync_temperature` + `set_target_temperature`).

**Logic:**
- **Valve position:** Computes `delta = target_temp − room_temp`, maps to 0–100% via a proportional lookup table, sends MQTT `valve_pos` command when position differs by ≥5% (hysteresis). Protects against boiler short-cycling.
- **Temperature sync:** Sends room temperature to the TRV via MQTT `ext_t` (overrides the TRV's internal sensor, which reads high due to valve motor heat).
- **Target temperature:** Sets TRV target via MQTT `target_t` when `input_number` changes. Supports window/door sensor for frost protection.

**Parameters:**

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `trv_entity` | ✅ | — | Climate entity of the Shelly TRV |
| `aqara_sensor` | ✅ | — | External room temperature sensor (Aqara or similar) |
| `trv_temperature_sensor` | ✅ | — | TRV's built-in temperature sensor |
| `current_valve_entity` | ✅ | — | `number.shelly_trv_*_valve_position` |
| `target_temp_input` | ✅ | — | `input_number` with desired room temperature |
| `mqtt_topic_base` | ✅ | — | Base MQTT topic (e.g. `shellies/shellytrv_bedroom_1/thermostat/0/command`) |
| `window_sensor` | ⬜ | `sun.sun` | Window/door sensor for frost protection |
| `frost_temp` | ⬜ | `8` | Frost temperature when window is open (°C) |
| `heating_status_sensor` | ⬜ | `sensor.heating_status` | Heating season sensor |
| `boiler_sensor` | ⬜ | `binary_sensor.protherm_24` | Boiler sensor for short-cycling protection |
| `window_close_delay` | ⬜ | `5` | Minutes to wait after window closes before restoring temperature |

---

### 2. Hydronic Boiler Controller

**File:** `blueprints/hydronic_boiler_controller.yaml`

Controls the boiler based on aggregate heating demand from all rooms. Handles both turn-on and turn-off logic in a single automation instance.

**Turn ON logic:**
- Triggers immediately when any thermostat sensor turns `on`
- Includes short-cycling protection: waits until the boiler has been stable for `min_cycle_protection` minutes before switching on
- Does not turn on outside the heating season

**Turn OFF logic:**
- Triggers after all thermostat sensors have been `off` for `turn_off_delay` minutes (delay is built into the trigger — no unnecessary pumping)
- If any thermostat turns on during the delay, the off-timer resets automatically (`mode: restart`)

**Parameters:**

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `boiler_switch` | ✅ | — | Switch entity controlling the boiler |
| `boiler_sensor` | ✅ | — | Binary sensor reflecting actual boiler state |
| `thermostat_sensors` | ✅ | — | List of `binary_sensor.thermostat_*` entities |
| `heating_status_sensor` | ⬜ | `sensor.heating_status` | Heating season sensor |
| `min_cycle_protection` | ⬜ | `10` | Minutes since last boiler state change before allowing turn-on |
| `turn_off_delay` | ⬜ | `10` | Minutes all thermostats must be off before turning off the boiler |

---

## Installation

See [docs/installation.md](docs/installation.md).

## Configuration Examples

See [docs/configuration.md](docs/configuration.md).

## Migrating from v1.x

See [docs/installation.md](docs/installation.md#migrating-from-v1x) for step-by-step migration instructions.

## Roadmap

See [docs/roadmap.md](docs/roadmap.md).

## License

MIT
