# Installation

## Step 1 — Create the Required Helper Entities

Follow [docs/prerequisites.md](prerequisites.md) to create:

1. `sensor.valve_position_*` — one per TRV
2. `binary_sensor.thermostat_*` — one per room
3. `sensor.heating_status` — one for the whole installation
4. `input_number.*_current` — one per room

Restart Home Assistant after adding the YAML template sensors.

---

## Step 2 — Install the Blueprints

Copy all `.yaml` files from the `blueprints/` directory to your Home Assistant blueprint folder:

```
/config/blueprints/automation/hydronic-heating/
```

Or install directly via the Home Assistant UI:

**Settings → Automations & Scenes → Blueprints → Import Blueprint**

Use the raw GitHub URL for each blueprint file.

---

## Step 3 — Create Automation Instances

Create one automation instance per blueprint per TRV, plus one Boiler Controller instance total:

| Blueprint | Instances |
|-----------|-----------|
| Set Valve Position | 1× per TRV |
| Sync External Temperature | 1× per TRV |
| Set Target Temperature | 1× per TRV |
| Boiler Controller | 1× total |

**Settings → Automations & Scenes → Create Automation → Use a Blueprint**

See [docs/configuration.md](configuration.md) for parameter examples.

---

## Step 4 — MQTT Topics

This system controls Shelly TRV devices via MQTT. The standard topic structure is:

```
shellies/<device-id>/thermostat/0/command/valve_pos   ← valve position (0-100)
shellies/<device-id>/thermostat/0/command/ext_t       ← external temperature
shellies/<device-id>/thermostat/0/command/target_t    ← target temperature
```

Find your device IDs in the Shelly app or from MQTT discovery in Home Assistant.

---

## Step 5 — Test

1. Raise the target temperature in one room above the current room temperature
2. Verify `sensor.valve_position_*` increases
3. Verify `number.shelly_trv_*_valve_position` follows (MQTT command sent)
4. Verify `binary_sensor.thermostat_*` turns `on` (valve > 18%)
5. Verify the boiler turns on
6. Lower the target temperature back
7. Verify the valve closes and the boiler turns off after `turn_off_delay` minutes
