# Heating Schedule System

This document explains how to set up and use the heating schedule system, which controls room temperatures based on time of day and presence.

---

## Overview

The heating schedule system is built around a central `input_select.heating_mode` entity. All temperature changes flow through this entity:

```
[Hydronic Heating Schedule]  ──► input_select.heating_mode
[Hydronic Presence Controller] ──►       │
                                         ▼
                               [heating_apply_mode automation]
                                         │
                              ┌──────────┴──────────┐
                              ▼                     ▼
                   input_number.*_current    ... (per room)
                              │
                              ▼
                   [shelly_trv_controller]
```

The `heating_apply_mode` automation translates the active mode into concrete target temperatures for each room. It is a plain automation — not a blueprint — because room names and entities are installation-specific.

---

## Step 1 — Create `input_select.heating_mode`

**Settings → Helpers → Add Helper → Select**

| Field | Value |
|-------|-------|
| Name | `Heating Mode` |
| Options | `Morning`, `Day`, `Evening`, `Night`, `Away`, `Holiday` |
| Icon | `mdi:home-thermometer` |

Verify the entity ID is `input_select.heating_mode`.

---

## Step 2 — Create per-room temperature helpers

For each room, create **5 `input_number` helpers** (Settings → Helpers → Add Helper → Number):

| Suffix | Purpose | Range |
|--------|---------|-------|
| `_morning` | Temperature 06:00–09:00 | 5–30°C, step 0.5 |
| `_day` | Temperature during the day | 5–30°C, step 0.5 |
| `_evening` | Temperature after sunset | 5–30°C, step 0.5 |
| `_night` | Temperature 22:00–06:00 | 5–30°C, step 0.5 |
| `_away_mode` | Temperature when nobody is home | 5–30°C, step 0.5 |

**Naming convention:** `input_number.heating_temperature_<room>_<suffix>`

**Example for a bedroom:**
- `input_number.heating_temperature_bedroom_morning`
- `input_number.heating_temperature_bedroom_day`
- `input_number.heating_temperature_bedroom_evening`
- `input_number.heating_temperature_bedroom_night`
- `input_number.heating_temperature_bedroom_away_mode`

You will also need one global holiday temperature:
- `input_number.heating_temperature_house_holiday_mode` — a single frost-protection temperature applied to all rooms when in Holiday mode (e.g. 15°C)

---

## Step 3 — Create `heating_apply_mode` automation

This automation watches `input_select.heating_mode` and sets each room's `*_current` entity to the corresponding temperature.

Add the following to your `automations.yaml`. Adjust the room list to match your installation:

```yaml
- id: heating_apply_mode
  alias: HVAC - Heating - Apply Mode
  description: Sets *_current temperatures based on the active heating mode
  triggers:
    - trigger: state
      entity_id: input_select.heating_mode
    # Re-apply when a person comes home (their room may need updating)
    - trigger: state
      entity_id: person.your_person
      to: home
    # Re-apply when TV state changes (Evening/Night TV logic, if applicable)
    # - trigger: state
    #   entity_id: media_player.your_tv
  actions:
    - choose:
        - conditions:
            - condition: state
              entity_id: input_select.heating_mode
              state: Morning
          sequence:
            - action: input_number.set_value
              data:
                value: "{{ states('input_number.heating_temperature_bedroom_morning') | float(0) }}"
              target:
                entity_id: input_number.heating_temperature_bedroom_current
            - action: input_number.set_value
              data:
                value: "{{ states('input_number.heating_temperature_kitchen_morning') | float(0) }}"
              target:
                entity_id: input_number.heating_temperature_kitchen_current
            # ... repeat for all rooms

        - conditions:
            - condition: state
              entity_id: input_select.heating_mode
              state: Day
          sequence:
            # ... day temperatures for all rooms

        - conditions:
            - condition: state
              entity_id: input_select.heating_mode
              state: Evening
          sequence:
            # ... evening temperatures for all rooms

        - conditions:
            - condition: state
              entity_id: input_select.heating_mode
              state: Night
          sequence:
            # ... night temperatures for all rooms

        - conditions:
            - condition: state
              entity_id: input_select.heating_mode
              state: Away
          sequence:
            - action: input_number.set_value
              data:
                value: "{{ states('input_number.heating_temperature_bedroom_away_mode') | float(0) }}"
              target:
                entity_id: input_number.heating_temperature_bedroom_current
            # ... away temperatures for all rooms

        - conditions:
            - condition: state
              entity_id: input_select.heating_mode
              state: Holiday
          sequence:
            - action: input_number.set_value
              data:
                value: "{{ states('input_number.heating_temperature_house_holiday_mode') | float(0) }}"
              target:
                entity_id:
                  - input_number.heating_temperature_bedroom_current
                  - input_number.heating_temperature_kitchen_current
                  # ... all rooms — one action, multiple targets
  mode: single
```

> **Per-person rooms:** If some rooms should only heat when a specific person is home (e.g. a child's room), wrap those actions in an `if` block:
> ```yaml
> - if:
>     - condition: state
>       entity_id: person.your_child
>       state: home
>   then:
>     - action: input_number.set_value
>       ...
> ```

---

## Step 4 — Install the blueprints

Install [Hydronic Heating Schedule](../blueprints/hydronic_heating_schedule.yaml) and [Hydronic Presence Controller](../blueprints/hydronic_presence_controller.yaml) and create one automation instance for each. See the [Installation guide](installation.md) for details.

---

## Boot sequence

After a Home Assistant restart, the system needs to restore the correct state. Both blueprints handle their own part automatically:

- **`hydronic_heating_schedule`** — fires at t+1 min, sets `input_select.heating_mode` to the correct time-based period (Morning/Day/Evening/Night)
- **`hydronic_presence_controller`** — fires at t+3 min, re-evaluates presence state and overrides to Away/Holiday if nobody is home

You still need a boot automation to run `heating_apply_mode` (which translates the mode into room temperatures) and to explicitly trigger your TRV controllers once — because if the temperature values didn't change since before the restart, the TRV controller's state trigger won't fire automatically.

```yaml
- id: heating_boot_sequence
  alias: HVAC - Heating - Boot Sequence
  trigger:
    - trigger: event
      event_type: homeassistant_started
  action:
    - delay:
        minutes: 2
    # heating_schedule has already set the correct mode at t+1 min
    - service: automation.trigger
      target:
        entity_id: automation.hvac_heating_apply_mode
      data:
        skip_condition: false
    - delay:
        minutes: 1
    # Force TRV controllers to run once and set valve positions
    - service: automation.trigger
      data:
        skip_condition: true
      target:
        entity_id:
          - automation.shelly_trv_bedroom_controller_v2_0
          - automation.shelly_trv_kitchen_controller_v2_0
          # ... add all your TRV controller automation entity IDs
```

**Boot timeline:**
```
t+1:00  heating_schedule     → sets correct time mode (Morning/Day/Evening/Night)
t+2:00  boot automation      → apply_mode (applies room temperatures)
t+3:00  boot automation      → TRV controllers (sets valve positions)
t+3:00  presence_controller  → overrides to Away/Holiday if nobody is home
```

---

## Hot water timer sync (optional)

If you have a Shelly 1 relay controlling a hot water boiler input, the physical switch input (`binary_sensor.*_input`) can mirror its state to the relay output:

```yaml
- id: hot_water_timer_sync
  alias: HVAC - Hot Water - Timer Sync
  trigger:
    - platform: state
      entity_id: binary_sensor.hot_water_stopper_input
  action:
    - if:
        - condition: state
          entity_id: binary_sensor.hot_water_stopper_input
          state: "on"
      then:
        - service: switch.turn_on
          target:
            entity_id: switch.hot_water_stopper
      else:
        - service: switch.turn_off
          target:
            entity_id: switch.hot_water_stopper
```

> **Note:** `switch.hot_water_stopper` uses inverted logic in this example — `ON` = heating stopped, `OFF` = heating active. Adjust to match your wiring.

---

## Dashboard card

```yaml
type: entities
title: Heating
entities:
  - entity: input_select.heating_mode
  - entity: sensor.heating_status
  - entity: binary_sensor.your_boiler_sensor
  - entity: switch.hot_water_stopper
    name: Hot water heating
```
