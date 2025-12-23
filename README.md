# HA-Tado-Holiday Override Mode

A Home Assistant Blueprint that provides a "Holiday Override" mode for Tado heating systems. This blueprint temporarily overlays your Tado heating schedules without permanently editing them, perfect for holidays or special occasions when you want to maintain comfort temperatures while people are home.

## Features

- **Non-Destructive Override**: Temporarily sets comfort temperatures without modifying your existing Tado schedules
- **People Presence Detection**: Only applies override when people are detected at home
- **Automatic Re-Application**: Periodically re-checks and re-applies temperature setpoints (default: every 30 minutes) to ensure Tado hasn't reverted them
- **Time Window Support**: Optional start and end times to restrict when the override can be active
- **Multi-Zone Control**: Control multiple Tado zones/rooms simultaneously
- **Easy Enable/Disable**: Simple boolean switch to turn the override on/off
- **Seamless Resume**: Normal Tado scheduling automatically resumes when override is disabled

## Installation

### Method 1: Import Blueprint URL

1. In Home Assistant, navigate to **Configuration** → **Blueprints**
2. Click the **Import Blueprint** button
3. Enter the following URL:
   ```
   https://github.com/Bibbleq/HA-Tado-Holiday_Mode/blob/main/blueprints/automation/tado_holiday_override.yaml
   ```
4. Click **Preview** and then **Import Blueprint**

### Method 2: Manual Installation

1. Copy the `tado_holiday_override.yaml` file to your Home Assistant configuration directory:
   ```
   config/blueprints/automation/tado_holiday_override/tado_holiday_override.yaml
   ```
2. Restart Home Assistant or reload automations

## Prerequisites

Before using this blueprint, ensure you have:

1. **Tado Integration**: The official Tado integration must be installed and configured in Home Assistant
2. **Input Boolean Helper**: Create an input boolean to act as the override switch
3. **Person/Presence Sensor**: A person entity, group, or binary sensor to detect presence

### Creating the Input Boolean Helper

1. Go to **Configuration** → **Helpers**
2. Click **Add Helper** → **Toggle**
3. Name it something like "Tado Holiday Override"
4. This will create an entity like `input_boolean.tado_holiday_override`

## Configuration

### Required Inputs

| Input | Description | Example |
|-------|-------------|---------|
| **Override Switch** | Boolean input to enable/disable the override | `input_boolean.tado_holiday_override` |
| **People Presence Sensor** | Entity to check if anyone is home | `person.john_doe`, `group.family`, `binary_sensor.someone_home` |
| **Tado Climate Zones** | Climate entities (rooms/zones) to control | `climate.tado_living_room`, `climate.tado_bedroom` |

### Optional Inputs

| Input | Description | Default | Range |
|-------|-------------|---------|-------|
| **Check Interval** | How often to re-apply temperatures (minutes) | 30 | 5-120 |
| **Comfort Temperature** | Target temperature when override is active | 21°C | 15-28°C |
| **Use Time Window** | Enable time-based restrictions | false | true/false |
| **Start Time** | When override can start being active | 00:00:00 | Any time |
| **End Time** | When override should stop being active | 23:59:59 | Any time |

## Usage Examples

### Example 1: Basic Holiday Override

Perfect for Christmas or other holidays when you want to keep the house warm while family is visiting.

**Configuration:**
- Override Switch: `input_boolean.tado_holiday_override`
- People Presence: `group.family`
- Tado Zones: `climate.tado_living_room`, `climate.tado_kitchen`, `climate.tado_dining_room`
- Comfort Temperature: `21°C`
- Check Interval: `30 minutes`

**Behavior:**
- Turn on the `input_boolean.tado_holiday_override` switch
- When anyone in `group.family` is home, all selected zones are set to 21°C
- Every 30 minutes, the automation re-checks and re-applies the temperature if needed
- Turn off the switch when you want to return to normal Tado schedules

### Example 2: Weekend Morning Comfort

Keep specific rooms warm during weekend mornings.

**Configuration:**
- Override Switch: `input_boolean.weekend_morning_override`
- People Presence: `binary_sensor.someone_home`
- Tado Zones: `climate.tado_bedroom`, `climate.tado_bathroom`
- Comfort Temperature: `22°C`
- Use Time Window: `true`
- Start Time: `07:00:00`
- End Time: `11:00:00`
- Check Interval: `15 minutes`

**Behavior:**
- Enable the override switch on Friday evening
- On Saturday and Sunday between 7 AM and 11 AM, if someone is home, bedrooms stay at 22°C
- Outside this time window, normal Tado schedules apply
- Every 15 minutes, temperatures are re-checked and re-applied

### Example 3: Work-From-Home Override

Maintain comfort in your home office during work hours.

**Configuration:**
- Override Switch: `input_boolean.wfh_override`
- People Presence: `person.me`
- Tado Zones: `climate.tado_office`
- Comfort Temperature: `20.5°C`
- Use Time Window: `true`
- Start Time: `09:00:00`
- End Time: `17:00:00`
- Check Interval: `45 minutes`

## How It Works

1. **Activation**: When you enable the override switch, the automation begins monitoring
2. **Condition Check**: Before applying temperatures, it verifies:
   - Override switch is ON
   - People are present at home
   - Current time is within the optional time window (if enabled)
3. **Temperature Application**: Sets all selected Tado zones to the comfort temperature
4. **Periodic Re-Check**: Every X minutes (configurable), the automation:
   - Re-evaluates all conditions
   - Re-applies the comfort temperature if conditions are still met
   - This ensures temperatures stay set even if Tado tries to revert to its schedule
5. **Deactivation**: When you turn off the override switch or the time window ends:
   - The automation stops enforcing temperatures
   - Tado resumes its normal programmed schedules
   - No permanent changes have been made to your Tado configuration

## Troubleshooting

### Temperatures keep reverting to schedule

- **Check interval too long**: Reduce the check interval (e.g., from 30 to 15 minutes)
- **Tado conflict mode**: Some Tado devices have a "conflict resolution" setting. Ensure it's compatible with external controls

### Override not activating

- **Verify switch state**: Check that the override switch entity is actually ON
- **Check presence sensor**: Ensure the presence entity shows the correct state ("home", "on", etc.)
- **Time window**: If using time windows, verify the current time is within the start/end range
- **Review logs**: Check Home Assistant logs for "Tado Holiday Override" messages

### Some zones not responding

- **Entity names**: Verify all climate entity IDs are correct
- **Tado connection**: Ensure the Tado integration is working properly
- **Preset mode support**: Some Tado devices may not support preset modes; the blueprint uses `continue_on_error` to handle this gracefully

## Advanced Customization

You can create multiple automations from this blueprint for different scenarios:

- **Holiday Override**: Higher temperatures for all main living areas
- **Guest Override**: Specific temperatures for guest bedroom and bathroom
- **Early Morning**: Warm bathroom before wake-up time
- **Evening Comfort**: Maintain temperature in living room during evenings

Each automation is independent and can have its own switch, zones, and settings.

## Technical Details

- **Mode**: `restart` - If triggered while running, it restarts to evaluate current conditions
- **Trigger Types**: 
  - State changes (override switch, presence sensor)
  - Time-based (periodic interval, start/end times)
- **Climate Controls**: 
  - Sets temperature via `climate.set_temperature`
  - Attempts to set preset mode to "home" (with error handling)
  - Includes small delays between zone updates to avoid overwhelming the Tado API

## Contributing

Issues and pull requests are welcome! Please report any bugs or suggest improvements on the [GitHub repository](https://github.com/Bibbleq/HA-Tado-Holiday_Mode).

## License

This project is open source and available under the MIT License.

## Credits

Created for the Home Assistant community to provide flexible, non-destructive temperature overrides for Tado heating systems.