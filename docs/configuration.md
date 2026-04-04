# Configuration Examples

This page shows example automation configurations for a typical multi-room setup.

---

## Set Valve Position — Example

```yaml
alias: Shelly TRV Bedroom 1 - Set Valve Position
use_blueprint:
  path: hydronic-heating/shelly_trv_set_valve_position.yaml
  input:
    trv_entity: climate.shelly_trv_bedroom_1
    valve_position_sensor: sensor.valve_position_bedroom_1
    current_valve_entity: number.shelly_trv_bedroom_1_valve_position
    aqara_sensor: sensor.aqara_t1_sensor_bedroom_temperature
    mqtt_topic: shellies/shellytrv_bedroom_1/thermostat/0/command/valve_pos
    # Optional — defaults shown:
    # heating_status_sensor: sensor.heating_status
    # boiler_sensor: binary_sensor.your_boiler_sensor
```

---

## Sync External Temperature — Example

```yaml
alias: Shelly TRV Bedroom 1 - Sync Temperature
use_blueprint:
  path: hydronic-heating/shelly_trv_sync_temperature.yaml
  input:
    aqara_sensor: sensor.aqara_t1_sensor_bedroom_temperature
    trv_temperature_sensor: sensor.shelly_trv_bedroom_1_temperature
    valve_position_sensor: sensor.valve_position_bedroom_1
    mqtt_topic: shellies/shellytrv_bedroom_1/thermostat/0/command/ext_t
    # Optional:
    # heating_status_sensor: sensor.heating_status
```

---

## Set Target Temperature — Example (with window sensor)

```yaml
alias: Shelly TRV Living Room - Set Target Temperature
use_blueprint:
  path: hydronic-heating/shelly_trv_set_target_temperature.yaml
  input:
    trv_entity: climate.shelly_trv_living_room_2
    target_temp_input: input_number.heating_temperature_living_room_current
    mqtt_topic: shellies/shellytrv_livingroom_2/thermostat/0/command/target_t
    window_sensor: binary_sensor.window_sensor_living_room_contact
    frost_temp: 8
```

## Set Target Temperature — Example (no window sensor)

```yaml
alias: Shelly TRV Corridor - Set Target Temperature
use_blueprint:
  path: hydronic-heating/shelly_trv_set_target_temperature.yaml
  input:
    trv_entity: climate.shelly_trv_corridor
    target_temp_input: input_number.heating_temperature_corridor_current
    mqtt_topic: shellies/shellytrv_corridor/thermostat/0/command/target_t
    # window_sensor omitted — defaults to sun.sun (never 'on')
```

---

## Boiler Turn ON — Example

```yaml
alias: Boiler - Turn ON
use_blueprint:
  path: hydronic-heating/hydronic_boiler_turn_on.yaml
  input:
    boiler_switch: switch.your_boiler_switch
    boiler_sensor: binary_sensor.your_boiler_sensor
    thermostat_sensors:
      - binary_sensor.thermostat_living_room
      - binary_sensor.thermostat_kitchen
      - binary_sensor.thermostat_bedroom
      - binary_sensor.thermostat_bathroom
      - binary_sensor.thermostat_corridor
    # Optional:
    # heating_status_sensor: sensor.heating_status
    # min_cycle_protection: 10
```

---

## Boiler Turn OFF — Example

```yaml
alias: Boiler - Turn OFF
use_blueprint:
  path: hydronic-heating/hydronic_boiler_turn_off.yaml
  input:
    boiler_switch: switch.your_boiler_switch
    thermostat_sensors:
      - binary_sensor.thermostat_living_room
      - binary_sensor.thermostat_kitchen
      - binary_sensor.thermostat_bedroom
      - binary_sensor.thermostat_bathroom
      - binary_sensor.thermostat_corridor
    # Optional:
    # turn_off_delay: 10
```

---

## Notes

- The `thermostat_sensors` list in Boiler Turn ON and Turn OFF must be identical.
- The `boiler_sensor` (binary sensor reflecting actual boiler state) is separate from `boiler_switch` (the switch you control). If your boiler switch IS the sensor, you can use the same entity for both.
- For rooms with multiple TRVs, create a combined `binary_sensor.thermostat_*` that ORs the individual sensors — see [prerequisites.md](prerequisites.md).
