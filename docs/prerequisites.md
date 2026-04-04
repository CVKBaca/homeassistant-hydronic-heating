# Prerequisites

Before using the blueprints, you need to create several supporting entities in Home Assistant. These are not included in the blueprints because they depend on your specific hardware (TRV model, temperature sensors) and heating schedule logic.

---

## 1. `sensor.valve_position_*` — Desired Valve Position

**Type:** UI Template Sensor (Home Assistant Helper)
**One per TRV**

This sensor computes the desired valve opening position (0–100%) from the temperature delta between the climate target temperature and the actual room temperature measured by an external sensor.

**How to create:** Settings → Helpers → Add Helper → Template → Sensor

**Template** (replace entity IDs to match your setup):

```jinja2
{% set delta = state_attr('climate.shelly_trv_bathroom', 'temperature') | float(0)
               - states('sensor.aqara_t1_sensor_bathroom_temperature') | float(25) %}
{% if delta <= -0.1 %}    0
{% elif delta <= -0.05 %} 13
{% elif delta <= 0 %}     18
{% elif delta <= 0.05 %}  25
{% elif delta <= 0.1 %}   33
{% elif delta <= 0.15 %}  40
{% elif delta <= 0.2 %}   48
{% elif delta <= 0.25 %}  55
{% elif delta <= 0.3 %}   63
{% elif delta <= 0.35 %}  70
{% elif delta <= 0.4 %}   78
{% elif delta <= 0.45 %}  85
{% elif delta <= 0.5 %}   93
{% else %}                100
{% endif %}
```

**Lookup table explained:**

| Delta (target − actual) | Valve position |
|-------------------------|----------------|
| ≤ −0.1°C (room too warm) | 0% |
| = 0°C (exactly at setpoint) | 18% |
| +0.2°C below setpoint | 48% |
| +0.5°C below setpoint | 93% |
| > +0.5°C below setpoint | 100% |

The 18% minimum when at setpoint keeps the TRV slightly open to maintain flow and avoid hunting.

> **Tip:** Adjust the lookup table to match your radiator sizing and room heat loss characteristics.

### Real-world example — 11 TRVs in one installation

The following shows the complete set of `sensor.valve_position_*` entities from a real installation with 11 Shelly TRVs and Aqara T1 temperature sensors:

| Entity | TRV climate entity | Aqara temperature sensor |
|--------|--------------------|--------------------------|
| `sensor.valve_position_bathroom` | `climate.shelly_trv_bathroom` | `sensor.aqara_t1_sensor_bathroom_temperature` |
| `sensor.valve_position_bedroom_1` | `climate.shelly_trv_bedroom_1` | `sensor.aqara_t1_sensor_bedroom_temperature` |
| `sensor.valve_position_bedroom_2` | `climate.shelly_trv_bedroom_2` | `sensor.aqara_t1_sensor_bedroom_temperature` |
| `sensor.valve_position_living_room_2` | `climate.shelly_trv_living_room_2` | `sensor.aqara_t1_sensor_living_room_temperature` |
| `sensor.valve_position_living_room_3` | `climate.shelly_trv_living_room_3` | `sensor.aqara_t1_sensor_living_room_temperature` |
| `sensor.valve_position_kitchen` | `climate.shelly_trv_kitchen` | `sensor.aqara_t1_sensor_kitchen_temperature` |
| `sensor.valve_position_corridor` | `climate.shelly_trv_corridor` | `sensor.aqara_t1_sensor_podesta_temperature` |
| `sensor.valve_position_hall` | `climate.shelly_trv_hall` | `sensor.aqara_t1_sensor_hall_temperature` |
| `sensor.valve_position_matej` | `climate.shelly_trv_matej` | `sensor.aqara_t1_sensor_matej_temperature` |
| `sensor.valve_position_richard` | `climate.shelly_trv_richard` | `sensor.aqara_t1_sensor_richard_temperature` |
| `sensor.valve_position_toilet` | `climate.shelly_trv_toilet` | `sensor.aqara_t1_sensor_toilet_temperature` |

Note: bedroom_1 and bedroom_2 share the same Aqara sensor — both TRVs use the same room temperature.
Living room TRVs 2 and 3 likewise share one sensor for the same room.

The template is identical for all — only the entity IDs differ:

```jinja2
{% set delta = state_attr('climate.shelly_trv_bedroom_1', 'temperature') | float(0)
               - states('sensor.aqara_t1_sensor_bedroom_temperature') | float(25) %}
{% if delta <= -0.1 %}    0
{% elif delta <= -0.05 %} 13
{% elif delta <= 0 %}     18
{% elif delta <= 0.05 %}  25
{% elif delta <= 0.1 %}   33
{% elif delta <= 0.15 %}  40
{% elif delta <= 0.2 %}   48
{% elif delta <= 0.25 %}  55
{% elif delta <= 0.3 %}   63
{% elif delta <= 0.35 %}  70
{% elif delta <= 0.4 %}   78
{% elif delta <= 0.45 %}  85
{% elif delta <= 0.5 %}   93
{% else %}                100
{% endif %}
```

---

## 2. `binary_sensor.thermostat_*` — Room Thermostat Sensor

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

> **Why 18%?** The valve position sensor returns 18% when delta = 0 (room is exactly at setpoint). Values above 18% indicate the room is below setpoint and actively heating.

---

## 3. `sensor.heating_status` — Heating Season Sensor

**Type:** UI Template Sensor (Home Assistant Helper)
**One per installation**

This sensor returns `On` or `Off` to indicate whether the heating system should be active at all. It is used by the Set Valve Position, Sync Temperature and Boiler Controller blueprints to suppress activity outside the heating season.

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

## 4. `input_number.*_current` — Target Temperature per Room

**Type:** Input Number Helper
**One per room**

Stores the desired target temperature for each room. Can be controlled from the UI, automations, or schedules.

**How to create:** Settings → Helpers → Add Helper → Number

- **Minimum:** 5°C
- **Maximum:** 30°C
- **Step:** 0.5°C

**Example entity IDs:**
- `input_number.heating_temperature_bedroom_current`
- `input_number.heating_temperature_living_room_current`
- `input_number.heating_temperature_bathroom_current`

---

## 5. Hardware: Permanently Open Radiator

⚠️ **At least one radiator in your hydraulic circuit must remain permanently open** (manually set to a fixed position, not controlled by this system).

**Why:** When all electronically controlled TRVs close, the boiler and circulation pump still run briefly before the Boiler Controller shuts them down (up to `turn_off_delay` minutes). During this time, the circuit needs an open path for water flow. Without it, pressure builds up and can damage the pump or boiler heat exchanger.

**Recommended:** Choose a radiator in a room that benefits from continuous low-level heating (hallway, bathroom) and leave its TRV fully open or at a fixed position.

---

## 6. Note on Condensing Boilers

If you have a **condensing gas or oil boiler**, the short-cycling protection (`min_cycle_protection`) is especially important. Condensing boilers:

- Require a minimum run time to reach operating temperature and enter condensing mode (maximum efficiency)
- Are more susceptible to wear from frequent short on/off cycles than conventional boilers
- Typically have built-in protection, but relying on it repeatedly shortens boiler lifespan

The default value of **10 minutes** is suitable for most condensing boilers. Consult your boiler manual for the manufacturer's recommended minimum run time.
