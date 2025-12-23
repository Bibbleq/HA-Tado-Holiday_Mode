# How It Works - Technical Overview

This document explains the technical implementation and logic flow of the Tado Holiday Override Mode blueprint.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Tado Holiday Override Mode                    │
│                                                                   │
│  Inputs:                                                          │
│  • Override Switch (input_boolean)                               │
│  • People Presence Sensor (person/group/binary_sensor)           │
│  • Tado Climate Zones (climate entities)                         │
│  • Comfort Temperature (°C)                                       │
│  • Optional: Time Window (start/end times)                        │
│  • Note: 30-minute check interval is fixed in the blueprint      │
└─────────────────────────────────────────────────────────────────┘
```

## Trigger Points

The automation can be triggered by five different events:

```
1. Override Switch State Change
   └─> User turns override ON or OFF

2. People Presence Change
   └─> Someone arrives home or leaves

3. Periodic Check (Time Pattern)
   └─> Runs every X minutes (default: 30)
   └─> Re-applies settings if conditions met

4. Time Window Start
   └─> Triggers at configured start time

5. Time Window End
   └─> Triggers at configured end time
```

## Logic Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                         Trigger Occurs                           │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
              ┌────────────────────────┐
              │ Check Override Switch   │
              │ State                   │
              └────────┬───────┬────────┘
                       │       │
               ┌───────┘       └───────┐
               │                       │
               ▼                       ▼
          ┌────────┐              ┌────────┐
          │  OFF   │              │   ON   │
          └────┬───┘              └────┬───┘
               │                       │
               ▼                       ▼
     ┌─────────────────┐    ┌─────────────────┐
     │ Stop Enforcement│    │ Check People     │
     │ Log: Disabled   │    │ Presence         │
     │ Exit            │    └────┬──────┬─────┘
     └─────────────────┘         │      │
                          ┌──────┘      └──────┐
                          │                    │
                          ▼                    ▼
                    ┌──────────┐         ┌──────────┐
                    │ Not Home │         │   Home   │
                    └─────┬────┘         └────┬─────┘
                          │                   │
                          ▼                   ▼
                    ┌──────────┐      ┌──────────────┐
                    │ Exit     │      │ Check Time   │
                    └──────────┘      │ Window       │
                                      └─────┬────┬───┘
                                            │    │
                                   ┌────────┘    └────────┐
                                   │                      │
                                   ▼                      ▼
                            ┌────────────┐        ┌────────────┐
                            │ Outside    │        │ Inside     │
                            │ Window     │        │ Window     │
                            └─────┬──────┘        └──────┬─────┘
                                  │                      │
                                  ▼                      ▼
                            ┌──────────┐          ┌─────────────┐
                            │ Exit     │          │ Apply Temps │
                            └──────────┘          │ to All Zones│
                                                  └──────┬──────┘
                                                         │
                                                         ▼
                                                  ┌──────────────┐
                                                  │ For Each Zone│
                                                  │ in Loop:     │
                                                  │              │
                                                  │ 1. Set Temp  │
                                                  │ 2. Set HVAC  │
                                                  │ 3. Set Preset│
                                                  │ 4. Wait 500ms│
                                                  └──────────────┘
```

## Action Logic Details

### Condition 1: Override OFF or Outside Time Window

```yaml
IF override_switch = OFF OR (time_window_enabled AND current_time NOT IN [start, end]):
  THEN:
    - Log: "Override disabled or outside time window"
    - Stop automation
    - Normal Tado scheduling resumes automatically
```

### Condition 2: Override ON, People Present, Inside Time Window

```yaml
IF override_switch = ON AND 
   people_presence = (home OR on) AND
   (NOT time_window_enabled OR current_time IN [start, end]):
  THEN:
    - Log: "Active - Setting N zones to X°C"
    - FOR EACH zone in tado_zones:
        - Set temperature to comfort_temperature
        - Set HVAC mode to "heat"
        - Try to set preset_mode to "home" (ignore errors)
        - Wait 500ms before next zone
```

### Default: Conditions Not Met

```yaml
ELSE:
  - Log: "Conditions not met for applying override" (debug level)
  - Exit without changes
```

## State Management

### Mode: Restart

```
mode: restart
```

- If automation triggers while already running, it **restarts**
- Ensures latest state is always evaluated
- Prevents multiple instances running simultaneously

### Max Exceeded: Silent

```
max_exceeded: silent
```

- If automation would exceed limits, fail silently
- Prevents log spam
- Ensures graceful handling of rapid triggers

## Variables

```yaml
zones: !input tado_zones           # List of climate entities
target_temp: !input comfort_temperature  # Target temperature (°C)
time_window_enabled: !input use_time_window  # Boolean flag
```

These variables are set once at automation start and used throughout the action sequence.

## Time Pattern Trigger Details

```yaml
platform: time_pattern
minutes: "/30"
```

This creates a repeating trigger that fires every 30 minutes:
- Fires at :00 and :30 of every hour
- Fixed interval - to change, fork the blueprint and modify the trigger
- Examples of other valid intervals: "/15" (every 15 min), "/60" (every hour)

## Climate Service Calls

### Set Temperature

```yaml
service: climate.set_temperature
target:
  entity_id: "{{ zones[repeat.index - 1] }}"
data:
  temperature: "{{ target_temp }}"
  hvac_mode: "heat"
```

- Sets target temperature
- Ensures HVAC mode is "heat"
- Applied to each zone individually

### Set Preset Mode

```yaml
service: climate.set_preset_mode
target:
  entity_id: "{{ zones[repeat.index - 1] }}"
data:
  preset_mode: "home"
continue_on_error: true
```

- Attempts to set Tado preset to "home"
- Uses `continue_on_error` because not all Tado devices support preset modes
- Fails gracefully if preset mode not supported

## Delay Between Zones

```yaml
delay:
  milliseconds: 500
```

- 500ms delay between processing each zone
- Prevents overwhelming the Tado API
- Reduces chance of rate limiting

## Time Window Logic

When `use_time_window: true`:

```
Time Check Logic:
- after: start_time
- before: end_time

Examples:
1. start: 07:00, end: 23:00
   → Active from 7 AM to 11 PM

2. start: 22:00, end: 08:00
   → Active from 10 PM to 8 AM (crosses midnight)

3. start: 00:00, end: 23:59
   → Active all day (essentially disabled)
```

## Logging

Three levels of logging are used:

### Info Level
```yaml
level: info
# Used when:
# - Override is disabled
# - Temperatures are being applied
```

### Debug Level
```yaml
level: debug
# Used when:
# - Conditions not met (to reduce log noise)
```

### Usage in Home Assistant
View logs at: **Settings** → **System** → **Logs**

Search for: `Tado Holiday Override`

## Performance Considerations

### API Call Frequency

Each execution cycle makes `N + 1` API calls where N = number of zones:
- N × `climate.set_temperature` calls
- N × `climate.set_preset_mode` calls (with error handling)

With default settings:
- 30-minute interval
- 4 zones
- = 8 API calls every 30 minutes
- = 384 API calls per day

### Optimization Tips

1. **Longer intervals** for less active monitoring:
   - 45-60 minutes for overnight periods
   - 15-30 minutes for active periods

2. **Time windows** to reduce unnecessary checks:
   - Only active during relevant hours
   - Saves API calls when not needed

3. **Zone grouping**:
   - Create separate automations for different zone groups
   - Different intervals for different priority areas

## Error Handling

### Robust Design

1. **Preset Mode**: `continue_on_error: true`
   - Won't fail if device doesn't support presets

2. **State Checks**: Multiple conditions
   - Validates all requirements before acting

3. **Restart Mode**: Self-correcting
   - Always evaluates current state
   - Recovers from transient issues

### Common Issues and Handling

| Issue | How Blueprint Handles It |
|-------|-------------------------|
| Tado offline | Service call fails, automation retries next interval |
| Wrong state type | OR conditions check multiple possible states (home/on) |
| Time window edge cases | Proper before/after logic handles midnight crossings |
| Rapid triggers | Restart mode prevents pileup |

## Integration Points

### Required Integrations
- **Tado** (climate entities)
- **Input Boolean** (helpers)

### Optional Integrations
- **Person** tracking
- **Zone** detection (home zone)
- **Group** entities (family tracking)
- **Binary Sensor** (presence detection)

## Extensibility

The blueprint can be extended by:

1. **Multiple Automations**:
   - Different zones, different temperatures
   - Different schedules, different triggers

2. **Condition Templates**:
   - Fork the blueprint
   - Add custom conditions (weather, energy price, etc.)

3. **Notification Integration**:
   - Add notification services in sequence
   - Alert when override activates/deactivates

## Best Practices

1. **Start Conservative**:
   - 30-minute interval initially
   - Monitor system load and adjust

2. **Test Thoroughly**:
   - Verify presence detection works
   - Confirm time windows behave as expected

3. **Monitor Logs**:
   - Check for errors first few days
   - Adjust based on actual behavior

4. **Document Your Setup**:
   - Note which zones use which automations
   - Record custom settings for future reference

## Technical Specifications

| Parameter | Value |
|-----------|-------|
| Blueprint Domain | automation |
| Mode | restart |
| Max Exceeded | silent |
| Minimum Interval | 5 minutes |
| Maximum Interval | 120 minutes |
| Temperature Range | 15-28°C |
| Temperature Step | 0.5°C |
| Inter-zone Delay | 500ms |

## Compatibility

### Home Assistant Version
- Minimum: 2023.1 (blueprint features)
- Recommended: 2024.x (latest stable)

### Tado Integration
- Official Tado integration required
- Climate entities must be available
- Preset mode support optional

### Entity Requirements
- Input Boolean: Required
- Climate: Required (1 or more)
- Presence: Required (person/group/binary_sensor/zone)
