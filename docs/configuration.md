# Configuration Examples

This page shows example automation configurations for a typical multi-room setup.

---

## Shelly TRV Controller — Example (with window sensor)

```yaml
alias: Shelly TRV Bedroom 1 - Controller
use_blueprint:
  path: hydronic-heating/shelly_trv_controller.yaml
  input:
    trv_entity: climate.shelly_trv_bedroom_1
    aqara_sensor: sensor.aqara_t1_sensor_bedroom_temperature
    trv_temperature_sensor: sensor.shelly_trv_bedroom_1_temperature
    current_valve_entity: number.shelly_trv_bedroom_1_valve_position
    target_temp_input: input_number.heating_temperature_bedroom_current
    mqtt_topic_base: shellies/shellytrv_bedroom_1/thermostat/0/command
    window_sensor: binary_sensor.bedroom_window_contact
    frost_temp: 8
    # Optional — defaults shown:
    # heating_status_sensor: sensor.heating_status
    # boiler_sensor: binary_sensor.your_boiler_sensor
    # hysteresis_threshold: 5
    # window_close_delay: 5
```

---

## Shelly TRV Controller — Example (no window sensor)

For rooms without a window/door sensor, omit `window_sensor` — it defaults to `sun.sun` which is never `on`.

```yaml
alias: Shelly TRV Corridor - Controller
use_blueprint:
  path: hydronic-heating/shelly_trv_controller.yaml
  input:
    trv_entity: climate.shelly_trv_corridor
    aqara_sensor: sensor.aqara_t1_sensor_podesta_temperature
    trv_temperature_sensor: sensor.shelly_trv_corridor_temperature
    current_valve_entity: number.shelly_trv_corridor_valve_position
    target_temp_input: input_number.heating_temperature_corridor_current
    mqtt_topic_base: shellies/shellytrv_corridor/thermostat/0/command
```

---

## Shelly TRV Controller — Real-world example (11 TRVs)

The following shows the complete automation configuration from a real 11-TRV installation.
Note that bedroom_1 and bedroom_2 share one Aqara temperature sensor, as do living_room_2 and living_room_3.

```yaml
# Bathroom
- alias: Shelly TRV Bathroom - Controller
  use_blueprint:
    path: hydronic-heating/shelly_trv_controller.yaml
    input:
      trv_entity: climate.shelly_trv_bathroom
      aqara_sensor: sensor.aqara_t1_sensor_bathroom_temperature
      trv_temperature_sensor: sensor.shelly_trv_bathroom_temperature
      current_valve_entity: number.shelly_trv_bathroom_valve_position
      target_temp_input: input_number.heating_temperature_bathroom_current
      mqtt_topic_base: shellies/shellytrv_bathroom/thermostat/0/command

# Bedroom 1 (shares Aqara sensor with Bedroom 2)
- alias: Shelly TRV Bedroom 1 - Controller
  use_blueprint:
    path: hydronic-heating/shelly_trv_controller.yaml
    input:
      trv_entity: climate.shelly_trv_bedroom_1
      aqara_sensor: sensor.aqara_t1_sensor_bedroom_temperature
      trv_temperature_sensor: sensor.shelly_trv_bedroom_1_temperature
      current_valve_entity: number.shelly_trv_bedroom_1_valve_position
      target_temp_input: input_number.heating_temperature_bedroom_current
      mqtt_topic_base: shellies/shellytrv_bedroom_1/thermostat/0/command
      window_sensor: binary_sensor.balcony_door

# Bedroom 2 (shares Aqara sensor with Bedroom 1)
- alias: Shelly TRV Bedroom 2 - Controller
  use_blueprint:
    path: hydronic-heating/shelly_trv_controller.yaml
    input:
      trv_entity: climate.shelly_trv_bedroom_2
      aqara_sensor: sensor.aqara_t1_sensor_bedroom_temperature
      trv_temperature_sensor: sensor.shelly_trv_bedroom_2_temperature
      current_valve_entity: number.shelly_trv_bedroom_2_valve_position
      target_temp_input: input_number.heating_temperature_bedroom_current
      mqtt_topic_base: shellies/shellytrv_bedroom_2/thermostat/0/command
      window_sensor: binary_sensor.balcony_door

# Living Room 2 (shares Aqara sensor with Living Room 3)
- alias: Shelly TRV Living Room 2 - Controller
  use_blueprint:
    path: hydronic-heating/shelly_trv_controller.yaml
    input:
      trv_entity: climate.shelly_trv_living_room_2
      aqara_sensor: sensor.aqara_t1_sensor_living_room_temperature
      trv_temperature_sensor: sensor.shelly_trv_living_room_2_temperature
      current_valve_entity: number.shelly_trv_living_room_2_valve_position
      target_temp_input: input_number.heating_temperature_living_room_current
      mqtt_topic_base: shellies/shellytrv_livingroom_2/thermostat/0/command

# Living Room 3 (shares Aqara sensor with Living Room 2)
- alias: Shelly TRV Living Room 3 - Controller
  use_blueprint:
    path: hydronic-heating/shelly_trv_controller.yaml
    input:
      trv_entity: climate.shelly_trv_living_room_3
      aqara_sensor: sensor.aqara_t1_sensor_living_room_temperature
      trv_temperature_sensor: sensor.shelly_trv_living_room_3_temperature
          current_valve_entity: number.shelly_trv_living_room_3_valve_position
      target_temp_input: input_number.heating_temperature_living_room_current
      mqtt_topic_base: shellies/shellytrv_livingroom_3/thermostat/0/command

# Kitchen
- alias: Shelly TRV Kitchen - Controller
  use_blueprint:
    path: hydronic-heating/shelly_trv_controller.yaml
    input:
      trv_entity: climate.shelly_trv_kitchen
      aqara_sensor: sensor.aqara_t1_sensor_kitchen_temperature
      trv_temperature_sensor: sensor.shelly_trv_kitchen_temperature
      current_valve_entity: number.shelly_trv_kitchen_valve_position
      target_temp_input: input_number.heating_temperature_kitchen_current
      mqtt_topic_base: shellies/shellytrv_kitchen/thermostat/0/command

# Corridor
- alias: Shelly TRV Corridor - Controller
  use_blueprint:
    path: hydronic-heating/shelly_trv_controller.yaml
    input:
      trv_entity: climate.shelly_trv_corridor
      aqara_sensor: sensor.aqara_t1_sensor_podesta_temperature
      trv_temperature_sensor: sensor.shelly_trv_corridor_temperature
      current_valve_entity: number.shelly_trv_corridor_valve_position
      target_temp_input: input_number.heating_temperature_corridor_current
      mqtt_topic_base: shellies/shellytrv_corridor/thermostat/0/command

# Hall
- alias: Shelly TRV Hall - Controller
  use_blueprint:
    path: hydronic-heating/shelly_trv_controller.yaml
    input:
      trv_entity: climate.shelly_trv_hall
      aqara_sensor: sensor.aqara_t1_sensor_hall_temperature
      trv_temperature_sensor: sensor.shelly_trv_hall_temperature
      current_valve_entity: number.shelly_trv_hall_valve_position
      target_temp_input: input_number.heating_temperature_hall_current
      mqtt_topic_base: shellies/shellytrv_hall/thermostat/0/command

# Toilet
- alias: Shelly TRV Toilet - Controller
  use_blueprint:
    path: hydronic-heating/shelly_trv_controller.yaml
    input:
      trv_entity: climate.shelly_trv_toilet
      aqara_sensor: sensor.aqara_t1_sensor_toilet_temperature
      trv_temperature_sensor: sensor.shelly_trv_toilet_temperature
      current_valve_entity: number.shelly_trv_toilet_valve_position
      target_temp_input: input_number.heating_temperature_toilet_current
      mqtt_topic_base: shellies/shellytrv_toilet/thermostat/0/command
```

---

## Boiler Controller — Example

A single automation handles both turn-on and turn-off:

```yaml
alias: Boiler Controller
use_blueprint:
  path: hydronic-heating/hydronic_boiler_controller.yaml
  input:
    boiler_switch: switch.your_boiler_switch
    boiler_sensor: binary_sensor.your_boiler_sensor
    thermostat_sensors:
      - binary_sensor.thermostat_living_room
      - binary_sensor.thermostat_kitchen
      - binary_sensor.thermostat_bedroom
      - binary_sensor.thermostat_bathroom
      - binary_sensor.thermostat_corridor
    # Optional — defaults shown:
    # heating_status_sensor: sensor.heating_status
    # min_cycle_protection: 10
    # turn_off_delay: 10
```

---

## Heating Schedule — Example

One automation instance controls the Morning/Day/Evening/Night time switching:

```yaml
alias: Heating Schedule
use_blueprint:
  path: hydronic-heating/hydronic_heating_schedule.yaml
  input:
    heating_mode_entity: input_select.heating_mode
    person_group: group.family
    morning_time: "06:00:00"
    day_time: "07:00:00"
    evening_offset: "-00:30:00"   # 30 minutes before sunset
    night_time: "22:00:00"
```

> **Evening offset:** A negative value triggers Evening mode before sunset, positive after. Make sure the computed sunset time stays between `day_time` and `night_time` — otherwise the Evening trigger is silently skipped.

---

## Presence Controller — Example

One automation instance handles Away/Holiday mode and domestic hot water:

```yaml
alias: Heating Presence Controller
use_blueprint:
  path: hydronic-heating/hydronic_presence_controller.yaml
  input:
    person_group: group.family
    heating_mode_entity: input_select.heating_mode
    hot_water_switch: switch.hot_water_stopper
    hot_water_input_sensor: binary_sensor.hot_water_stopper_input
    holiday_threshold_hours: 18
```

See [presence-control.md](presence-control.md) for hot water wiring details and how to create a dummy `hot_water_input_sensor` if you do not have a physical override input.

---

## Notes

- The `thermostat_sensors` list must cover the same rooms as the TRV controller instances. If a room is omitted from `thermostat_sensors`, the boiler will never turn on for that room.
- The `boiler_sensor` (binary sensor reflecting actual boiler state) is separate from `boiler_switch` (the switch you control). If your boiler switch IS the sensor, you can use the same entity for both.
- For rooms with multiple TRVs sharing one Aqara sensor, both TRV controller instances use the same `aqara_sensor` entity. This is fine — the blueprint handles it correctly.
- For rooms with multiple TRVs, create a combined `binary_sensor.thermostat_*` that ORs the individual sensors — see [prerequisites.md](prerequisites.md).
