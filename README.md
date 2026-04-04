# Hydronic Heating for Home Assistant

A set of Home Assistant blueprints for controlling hydronic (hot water) heating systems with **Shelly TRV** thermostatic radiator valves and a gas/oil boiler.

The system implements **proportional valve control** — instead of simple on/off, each radiator valve opens proportionally to the temperature deficit. The boiler is controlled automatically based on actual heating demand.

---

## ⚠️ Important Safety Requirement

**At least one radiator in the hydraulic circuit must remain permanently open** (manually set, not controlled by this system).

If all radiators close simultaneously, the boiler and circulation pump will push water into a closed circuit — this can damage or destroy your equipment.

---

## System Architecture

```
[Aqara temp sensor] ──► [sensor.valve_position_*] ──► [Set Valve Position blueprint]
[climate target temp] ──►                                        │
                                                                  ▼
[Aqara temp sensor] ──────────────────────────────► [Sync Temperature blueprint]
                                                                  │
[input_number target] ─────────────────────────────► [Set Target Temp blueprint]
                                                                  │
                                                    [MQTT → Shelly TRV]
                                                                  │
                                              [number.shelly_trv_*_valve_position]
                                                                  │
                                              [binary_sensor.thermostat_*] (>18%)
                                                                  │
                                    ┌─────────────────────────────┘
                                    │
                         [Boiler Turn ON blueprint] ──► [boiler switch]
                         [Boiler Turn OFF blueprint] ──►
```

---

## Prerequisites

Before installing the blueprints, you need to create the following entities manually. See [docs/prerequisites.md](docs/prerequisites.md) for detailed instructions.

| Entity | Type | Description |
|--------|------|-------------|
| `sensor.valve_position_*` | UI Template Sensor | Computes desired valve position (%) from temperature delta |
| `binary_sensor.thermostat_*` | YAML Template Sensor | `true` when valve position > 18% (room needs heat) |
| `sensor.heating_status` | UI Template Sensor | Returns `On`/`Off` based on outdoor temperature / season |
| `input_number.*_current` | Input Number | Target temperature per room, controllable from UI |

---

## Blueprints

### 1. Shelly TRV — Set Valve Position

**File:** `blueprints/shelly_trv_set_valve_position.yaml`

Controls the valve opening position based on the temperature delta between the climate target temperature and the actual room temperature (from an external Aqara sensor).

**Logic:**
- Computes `delta = target_temp − room_temp`
- Maps delta to valve position via a proportional lookup table (0–100%)
- Sends MQTT command only when actual position differs by ≥5% (hysteresis)
- Protects against boiler short-cycling: waits if the boiler recently changed state

**Parameters:**

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `trv_entity` | ✅ | — | Climate entity of the Shelly TRV |
| `valve_position_sensor` | ✅ | — | `sensor.valve_position_*` for this TRV |
| `current_valve_entity` | ✅ | — | `number.shelly_trv_*_valve_position` |
| `aqara_sensor` | ✅ | — | External temperature sensor |
| `mqtt_topic` | ✅ | — | MQTT topic for `valve_pos` command |
| `heating_status_sensor` | ⬜ | `sensor.heating_status` | Heating season sensor |
| `boiler_sensor` | ⬜ | `binary_sensor.protherm_24` | Boiler binary sensor for short-cycling protection |

---

### 2. Shelly TRV — Sync External Temperature

**File:** `blueprints/shelly_trv_sync_temperature.yaml`

Sends the external Aqara room temperature to the Shelly TRV via MQTT (`ext_t` command). This overrides the TRV's internal sensor, which is less accurate due to heat from the valve motor.

**Logic:**
- Triggers on every Aqara sensor state change
- Runs only when the valve is open (> 0%) and heating is active
- If the TRV's own sensor changed recently and differs by > 0.04°C, waits up to 5 minutes for it to stabilise

**Parameters:**

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `aqara_sensor` | ✅ | — | External temperature sensor |
| `trv_temperature_sensor` | ✅ | — | TRV's built-in temperature sensor |
| `valve_position_sensor` | ✅ | — | `sensor.valve_position_*` for this TRV |
| `mqtt_topic` | ✅ | — | MQTT topic for `ext_t` command |
| `heating_status_sensor` | ⬜ | `sensor.heating_status` | Heating season sensor |

---

### 3. Shelly TRV — Set Target Temperature

**File:** `blueprints/shelly_trv_set_target_temperature.yaml`

Sets the TRV target temperature via MQTT when the corresponding `input_number` changes. Optionally supports a window/door sensor — when the window opens, sets frost protection temperature; when it closes, restores normal temperature after 5 minutes.

**Parameters:**

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `trv_entity` | ✅ | — | Climate entity of the Shelly TRV |
| `target_temp_input` | ✅ | — | `input_number` with desired temperature |
| `mqtt_topic` | ✅ | — | MQTT topic for `target_t` command |
| `window_sensor` | ⬜ | `sun.sun` | Window/door binary sensor. Leave default if none. |
| `frost_temp` | ⬜ | `8` | Frost protection temperature (°C) when window is open |

---

### 4. Hydronic Boiler — Turn ON

**File:** `blueprints/hydronic_boiler_turn_on.yaml`

Turns on the boiler when at least one thermostat sensor reports heating demand. Includes short-cycling protection: waits until the boiler has been in its current state for a minimum time before switching on.

**Parameters:**

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `boiler_switch` | ✅ | — | Switch entity controlling the boiler |
| `boiler_sensor` | ✅ | — | Binary sensor reflecting actual boiler state |
| `thermostat_sensors` | ✅ | — | List of `binary_sensor.thermostat_*` entities |
| `heating_status_sensor` | ⬜ | `sensor.heating_status` | Heating season sensor |
| `min_cycle_protection` | ⬜ | `10` | Minutes since last boiler state change before allowing turn-on |

---

### 5. Hydronic Boiler — Turn OFF

**File:** `blueprints/hydronic_boiler_turn_off.yaml`

Turns off the boiler after all thermostat sensors have been off for a configurable delay. The delay is built into the trigger — no unnecessary pumping after all valves close.

**Parameters:**

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `boiler_switch` | ✅ | — | Switch entity controlling the boiler |
| `thermostat_sensors` | ✅ | — | List of `binary_sensor.thermostat_*` entities |
| `turn_off_delay` | ⬜ | `10` | Minutes all thermostats must be off before turning off the boiler |

---

## Installation

See [docs/installation.md](docs/installation.md).

## Configuration Examples

See [docs/configuration.md](docs/configuration.md).

## License

MIT
