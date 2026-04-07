# Presence Control

This document explains how the Hydronic Presence Controller blueprint handles Away and Holiday modes, hot water control, and automatic mode restoration when the household returns home.

---

## Overview

The `Hydronic Presence Controller` blueprint monitors a `group` entity (your household members) and reacts to presence changes:

| Event | Action |
|-------|--------|
| Everyone leaves | → Away mode + hot water OFF |
| Away for 18h+ | → Holiday mode |
| Someone returns | → Hot water ON + correct mode based on time |
| 06:00 (people home) | → Hot water ON (re-enable after midnight off) |
| 00:00 | → Hot water OFF (overnight safety fallback) |

---

## Step 1 — Create a person group

**Settings → Helpers → Add Helper → Group → Person group**

Add all household members. The group state is `home` when at least one person is home, `not_home` when everyone is away.

> Alternatively, create the group in YAML:
> ```yaml
> group:
>   person:
>     name: Family
>     entities:
>       - person.alice
>       - person.bob
> ```

---

## Step 2 — Hot water switch wiring

The blueprint controls hot water via a `switch` entity. The expected wiring matches a Shelly 1 relay connected to the boiler's DHW (domestic hot water) input:

| Switch state | Meaning |
|-------------|---------|
| `ON` (relay active) | Hot water heating **stopped** |
| `OFF` (relay inactive) | Hot water heating **active** |

> **This is inverted logic.** If your wiring is the opposite, swap `turn_on` and `turn_off` in the blueprint, or invert the switch entity using a template.

The blueprint also reads a `binary_sensor` reflecting the physical input of the relay (e.g. `binary_sensor.hot_water_stopper_input` from a Shelly 1). When this sensor is `on`, the automation assumes hot water was manually overridden and will not turn it back on automatically.

If you do not have a physical override input, create a dummy binary sensor that always returns `off`:

```yaml
# config/templates/binary_sensors.yaml
- binary_sensor:
    - name: "Hot water manual override"
      unique_id: "hot_water_manual_override"
      state: "false"
```

---

## Step 3 — Install the blueprint

See [Installation guide](installation.md).

**Blueprint parameters:**

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `person_group` | ✅ | — | `group` entity for household presence |
| `heating_mode_entity` | ✅ | — | `input_select.heating_mode` |
| `hot_water_switch` | ✅ | — | Switch controlling hot water (inverted: ON = stopped) |
| `hot_water_input_sensor` | ✅ | — | Binary sensor for physical override input |
| `holiday_threshold_hours` | ⬜ | `18` | Hours away before switching to Holiday mode |

---

## How modes work

### Away mode
Activates immediately when everyone leaves. Sets mode to `Away` and turns off hot water heating. Your `heating_apply_mode` automation should react to this mode change and apply the per-room away temperatures (`input_number.*_away_mode`).

### Holiday mode
Activates after the household has been away for `holiday_threshold_hours` (default: 18 hours). Typically uses a single frost-protection temperature for the whole house (`input_number.heating_temperature_house_holiday_mode`).

The distinction between Away and Holiday is deliberate:
- **Away** — short absence, keep rooms reasonably warm for quick return
- **Holiday** — extended absence, minimal frost-protection temperatures save energy

### Return home
When someone arrives, the blueprint:
1. Turns on hot water (unless manually overridden)
2. Sets the heating mode based on the current time of day:
   - 06:00–07:00 → Morning
   - 07:00–sunset → Day
   - sunset–22:00 → Evening
   - 22:00–06:00 → Night

---

## Interaction with the Heating Schedule blueprint

Both blueprints can set `input_select.heating_mode`. They do not conflict:

- **Heating Schedule** sets Morning/Day/Evening/Night based on clock and sun
- **Presence Controller** overrides to Away/Holiday when nobody is home, and restores a time-appropriate mode on return

When someone returns home, the Presence Controller sets a time-appropriate mode, and the Heating Schedule will continue switching modes normally from its next trigger onwards.

---

## Boot behaviour

After a Home Assistant restart, the Presence Controller automatically re-evaluates the household presence state at **t+3 minutes**:

| State at boot | Action |
|---------------|--------|
| Nobody home, away < `holiday_threshold_hours` | → Away mode + hot water OFF |
| Nobody home, away ≥ `holiday_threshold_hours` | → Holiday mode + hot water OFF |
| Someone home, hot water input sensor OFF | → Hot water ON (no mode change — Heating Schedule handles the mode) |

The 3-minute delay ensures the Heating Schedule blueprint has already set the correct time-based mode first (t+1 min), and `heating_apply_mode` has run (t+2 min), before the Presence Controller potentially overrides to Away/Holiday.
