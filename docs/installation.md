# Installation

## Step 1 — Create the Required Helper Entities

Follow [prerequisites.md](prerequisites.md) to create:

1. `binary_sensor.thermostat_*` — one per room (YAML template sensors)
2. `sensor.heating_status` — one for the whole installation
3. `input_number.*_current` — one per room

Restart Home Assistant after adding the YAML template sensors.

> **Note:** `sensor.valve_position_*` is **not required** in v2.0. The valve position lookup table is now computed inside the `shelly_trv_controller` blueprint.

---

## Step 2 — Install the Blueprints

Copy the `.yaml` files from the `blueprints/` directory to your Home Assistant blueprint folder:

```
/config/blueprints/automation/hydronic-heating/
    shelly_trv_controller.yaml
    hydronic_boiler_controller.yaml
```

Or install directly via the Home Assistant UI:

**Settings → Automations & Scenes → Blueprints → Import Blueprint**

Use the raw GitHub URL for each blueprint file.

---

## Step 3 — Create Automation Instances

Create one `shelly_trv_controller` instance per TRV, plus one `hydronic_boiler_controller` instance total:

| Blueprint | Instances |
|-----------|-----------|
| `shelly_trv_controller` | 1× per TRV |
| `hydronic_boiler_controller` | 1× total |

**Settings → Automations & Scenes → Create Automation → Use a Blueprint**

See [configuration.md](configuration.md) for parameter examples.

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

---

## Migrating from v1.x

If you are migrating from the 3-blueprint system (`set_valve_position` + `sync_temperature` + `set_target_temperature`):

1. Install `shelly_trv_controller.yaml` blueprint
2. Create new automation instances (one per TRV)
3. Run alongside v1.x automations for 2–3 days (disable, not delete)
4. If stable: delete old 3× automation instances per TRV + old 3 blueprints
5. Delete `sensor.valve_position_*` UI helpers — no longer needed
