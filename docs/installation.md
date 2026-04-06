# Installation

## Step 1 — Create the Required Helper Entities

Follow [prerequisites.md](prerequisites.md) to create:

1. `binary_sensor.thermostat_*` — one per room (YAML template sensors)
2. `sensor.heating_status` — one for the whole installation
3. `input_number.*_current` — one per room

Restart Home Assistant after adding the YAML template sensors.

---

## Step 2 — Install the Blueprints

Copy the `.yaml` files from the `blueprints/` directory to your Home Assistant blueprint folder:

```
/config/blueprints/automation/hydronic-heating/
    shelly_trv_controller.yaml
    hydronic_boiler_controller.yaml
    hydronic_heating_schedule.yaml
    hydronic_presence_controller.yaml
```

Or install directly via the Home Assistant UI:

**Settings → Automations & Scenes → Blueprints → Import Blueprint**

Use the raw GitHub URL for each blueprint file.

---

## Step 3 — Create Automation Instances

| Blueprint | Instances | Purpose |
|-----------|-----------|---------|
| `shelly_trv_controller` | 1× per TRV | Valve control, temperature sync, target temperature |
| `hydronic_boiler_controller` | 1× total | Boiler on/off with short-cycling protection |
| `hydronic_heating_schedule` | 1× total | Morning/Day/Evening/Night switching by time + sun |
| `hydronic_presence_controller` | 1× total | Away/Holiday mode + hot water by presence |

**Settings → Automations & Scenes → Create Automation → Use a Blueprint**

See [configuration.md](configuration.md) for parameter examples.

Before creating automation instances for `hydronic_heating_schedule` and `hydronic_presence_controller`, complete Steps 4–6 in [heating-schedule.md](heating-schedule.md) (create `input_select.heating_mode`, per-room temperature helpers, and the `heating_apply_mode` plain automation).

---

## Step 4 — MQTT Topics

This system controls Shelly TRV devices via MQTT. The standard topic structure is:

```
shellies/<device-id>/thermostat/0/command/valve_pos   ← valve position (0–100)
shellies/<device-id>/thermostat/0/command/ext_t       ← external temperature
shellies/<device-id>/thermostat/0/command/target_t    ← target temperature
```

In the blueprint you provide the **base topic** (without the suffix):

```
shellies/<device-id>/thermostat/0/command
```

The blueprint adds `/valve_pos`, `/ext_t`, and `/target_t` automatically.

Find your device IDs in the Shelly app or from MQTT discovery in Home Assistant.

---

## Step 5 — Test

1. Raise the target temperature (`input_number.*_current`) in one room above the current room temperature
2. Verify the valve position increases (check automation trace or `number.shelly_trv_*_valve_position`)
3. Verify `binary_sensor.thermostat_*` turns `on` (valve > 18%)
4. Verify the boiler turns on
5. Lower the target temperature back
6. Verify the valve closes and the boiler turns off after `turn_off_delay` minutes

