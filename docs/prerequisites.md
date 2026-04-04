# Prerequisites

Before using the blueprints, you need to create several supporting entities in Home Assistant. These are not included in the blueprints because they depend on your specific hardware (TRV model, temperature sensors) and heating schedule logic.

---

## 1. `binary_sensor.thermostat_*` — Room Thermostat Sensor

**Type:** YAML Template Sensor (`config/templates/`)  
**One per room (or one combining multiple TRVs in the same room)**

This sensor is `on` when a room is actively heating (valve position > 18%). It is used by the Boiler Controller blueprint to determine heating demand.

**How to create:** Add to your `configuration.yaml` or a file included via `template: !include_dir_merge_list`:

```yaml
# config/templates/binary_sensors.yaml
- binary_sensor:
    # Simple room with one TRV:
    - name: "Thermostat - Kitchen"
      unique_id: "thermostat_kitchen"
      state: >-
        {{ states('number.shelly_trv_kitchen_valve_position') | int(0) > 18 }}

    # Room with multiple TRVs — combine with OR:
    - name: "Thermostat - Bedroom"
      unique_id: "thermostat_bedroom"
      state: >-
        {{ states('binary_sensor.thermostat_bedroom_1') == 'on'
           or states('binary_sensor.thermostat_bedroom_2') == 'on' }}

    - name: "Thermostat - Bedroom 1"
      unique_id: "thermostat_bedroom_1"
      state: >-
        {{ states('number.shelly_trv_bedroom_1_valve_position') | int(0) > 18 }}

    - name: "Thermostat - Bedroom 2"
      unique_id: "thermostat_bedroom_2"
      state: >-
        {{ states('number.shelly_trv_bedroom_2_valve_position') | int(0) > 18 }}
```

> **Why 18%?** The `shelly_trv_controller` blueprint returns 18% when the room is exactly at setpoint (delta = 0°C). Values above 18% indicate the room is below setpoint and actively calling for heat.

### Real-world example — 11 TRVs in one installation

```yaml
- binary_sensor:
    - name: "Thermostat - Bathroom"
      unique_id: "thermostat_bathroom"
      state: >-
        {{ states('number.shelly_trv_bathroom_valve_position') | int(0) > 18 }}

    - name: "Thermostat - Bedroom 1"
      unique_id: "thermostat_bedroom_1"
      state: >-
        {{ states('number.shelly_trv_bedroom_1_valve_position') | int(0) > 18 }}

    - name: "Thermostat - Bedroom 2"
      unique_id: "thermostat_bedroom_2"
      state: >-
        {{ states('number.shelly_trv_bedroom_2_valve_position') | int(0) > 18 }}

    # Bedroom combines Bedroom 1 + Bedroom 2
    - name: "Thermostat - Bedroom"
      unique_id: "thermostat_bedroom"
      state: >-
        {{ states('binary_sensor.thermostat_bedroom_1') == 'on'
           or states('binary_sensor.thermostat_bedroom_2') == 'on' }}

    - name: "Thermostat - Living Room 2"
      unique_id: "thermostat_living_room_2"
      state: >-
        {{ states('number.shelly_trv_living_room_2_valve_position') | int(0) > 18 }}

    - name: "Thermostat - Living Room 3"
      unique_id: "thermostat_living_room_3"
      state: >-
        {{ states('number.shelly_trv_living_room_3_valve_position') | int(0) > 18 }}

    # Living Room combines LR2 + LR3
    - name: "Thermostat - Living Room"
      unique_id: "thermostat_living_room"
      state: >-
        {{ states('binary_sensor.thermostat_living_room_2') == 'on'
           or states('binary_sensor.thermostat_living_room_3') == 'on' }}

    - name: "Thermostat - Kitchen"
      unique_id: "thermostat_kitchen"
      state: >-
        {{ states('number.shelly_trv_kitchen_valve_position') | int(0) > 18 }}

    - name: "Thermostat - Corridor"
      unique_id: "thermostat_corridor"
      state: >-
        {{ states('number.shelly_trv_corridor_valve_position') | int(0) > 18 }}

    - name: "Thermostat - Hall"
      unique_id: "thermostat_hall"
      state: >-
        {{ states('number.shelly_trv_hall_valve_position') | int(0) > 18 }}

    - name: "Thermostat - Matej"
      unique_id: "thermostat_matej"
      state: >-
        {{ states('number.shelly_trv_matej_valve_position') | int(0) > 18 }}

    - name: "Thermostat - Richard"
      unique_id: "thermostat_richard"
      state: >-
        {{ states('number.shelly_trv_richard_valve_position') | int(0) > 18 }}

    - name: "Thermostat - Toilet"
      unique_id: "thermostat_toilet"
      state: >-
        {{ states('number.shelly_trv_toilet_valve_position') | int(0) > 18 }}
```

After adding these sensors, restart Home Assistant.

---

## 2. `sensor.heating_status` — Heating Season Sensor

**Type:** UI Template Sensor (Home Assistant Helper)  
**One per installation**

This sensor returns `On` or `Off` to indicate whether the heating system should be active at all. It is used by the TRV Controller and Boiler Controller blueprints to suppress activity outside the heating season.

**How to create:** Settings → Helpers → Add Helper → Template → Sensor

**Template example** (season + outdoor temperature with fallback):

```jinja2
{% set month = now().month %}
{% set outside = states('sensor.your_outdoor_temperature_sensor') %}
{% if month in [10, 11, 12, 1, 2, 3] %}
  On
{% elif outside not in ['unavailable', 'unknown'] %}
  {{ 'On' if outside | float(99) < 13 else 'Off' }}
{% else %}
  Off
{% endif %}
```

**Logic explained:**
- **October–March**: heating always `On` regardless of outdoor temperature
- **April–September**: heating `On` only if outdoor temperature < 13°C
- **Fallback**: if outdoor sensor is unavailable, heating is `Off`

> **Tip:** Adjust the month range and temperature threshold to match your climate and personal preferences.

---

## 3. `input_number.*_current` — Target Temperature per Room

**Type:** Input Number Helper  
**One per room**

Stores the desired target temperature for each room. The TRV Controller blueprint reads this value to determine the target temperature and compute the valve position.

**How to create:** Settings → Helpers → Add Helper → Number

- **Minimum:** 5°C
- **Maximum:** 30°C
- **Step:** 0.5°C

**Naming convention:** `input_number.heating_temperature_<room>_current`

**Example entity IDs:**

| Room | Entity ID |
|------|-----------|
| Bathroom | `input_number.heating_temperature_bathroom_current` |
| Bedroom | `input_number.heating_temperature_bedroom_current` |
| Living Room | `input_number.heating_temperature_living_room_current` |
| Kitchen | `input_number.heating_temperature_kitchen_current` |
| Corridor | `input_number.heating_temperature_corridor_current` |
| Hall | `input_number.heating_temperature_hall_current` |
| Toilet | `input_number.heating_temperature_toilet_current` |

> **Tip:** These values can be set by automations (e.g. a heating schedule that adjusts temperature by time of day) or manually from the UI. The TRV controller reacts within 15 minutes at most (safety net trigger), or immediately when the value changes.

---

## 4. Hardware: Permanently Open Radiator

⚠️ **At least one radiator in your hydraulic circuit must remain permanently open** (manually set to a fixed position, not controlled by this system).

**Why:** When all electronically controlled TRVs close, the boiler and circulation pump still run briefly before the Boiler Controller shuts them down (up to `turn_off_delay` minutes). During this time, the circuit needs an open path for water flow. Without it, pressure builds up and can damage the pump or boiler heat exchanger.

**Recommended:** Choose a radiator in a room that benefits from continuous low-level heating (hallway, bathroom) and leave its TRV fully open or at a fixed position.

---

## 5. Note on Condensing Boilers

If you have a **condensing gas or oil boiler**, the short-cycling protection (`min_cycle_protection`) is especially important. Condensing boilers:

- Require a minimum run time to reach operating temperature and enter condensing mode (maximum efficiency)
- Are more susceptible to wear from frequent short on/off cycles than conventional boilers
- Typically have built-in protection, but relying on it repeatedly shortens boiler lifespan

The default value of **10 minutes** is suitable for most condensing boilers. Consult your boiler manual for the manufacturer's recommended minimum run time.

---

## What's NOT required in v2.0

`sensor.valve_position_*` — these UI helper sensors were required in v1.x but are **no longer needed** in v2.0. The valve position lookup table is now computed internally by the `shelly_trv_controller` blueprint. If you are migrating from v1.x, you can delete these helpers after switching to the new blueprint.
