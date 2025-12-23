# Quick Start Guide

Get up and running with Tado Holiday Override Mode in 5 minutes!

## Step 1: Create the Override Switch

1. In Home Assistant, go to **Settings** → **Devices & Services** → **Helpers**
2. Click **+ Create Helper** → **Toggle**
3. Fill in the details:
   - **Name**: `Tado Holiday Override`
   - **Icon**: `mdi:home-thermometer` (optional)
4. Click **Create**
5. Note the entity ID: `input_boolean.tado_holiday_override`

## Step 2: Import the Blueprint

### Option A: Import via URL (Easiest)
1. Go to **Settings** → **Automations & Scenes** → **Blueprints**
2. Click **Import Blueprint** (blue button in bottom right)
3. Paste this URL:
   ```
   https://github.com/Bibbleq/HA-Tado-Holiday_Mode/blob/main/blueprints/automation/tado_holiday_override.yaml
   ```
4. Click **Preview Blueprint** → **Import Blueprint**

### Option B: Manual Installation
1. Download `tado_holiday_override.yaml` from this repository
2. Place it in: `config/blueprints/automation/tado_holiday_override/tado_holiday_override.yaml`
3. Restart Home Assistant

## Step 3: Create an Automation

1. Go to **Settings** → **Automations & Scenes** → **Automations**
2. Click **+ Create Automation** → **Use a blueprint**
3. Select **Tado Holiday Override Mode**
4. Fill in the required fields:

### Minimal Configuration
```yaml
Override Switch: input_boolean.tado_holiday_override
People Presence Sensor: person.your_name (or group.family)
Tado Climate Zones: 
  - climate.tado_living_room
  - climate.tado_bedroom
Comfort Temperature: 21
```

5. Click **Save** and give it a name like "Holiday Override"

## Step 4: Test It!

1. Go to **Settings** → **Devices & Services** → **Helpers**
2. Find your `Tado Holiday Override` toggle
3. Turn it **ON**
4. Check your Tado zones - they should now be set to your comfort temperature!
5. Turn it **OFF** to return to normal Tado scheduling

## What Happens Next?

- **Every 30 minutes** (or your chosen interval), the automation re-checks:
  - Is the override switch ON?
  - Is someone home?
  - Is it within the time window (if configured)?
- If all conditions are met, it re-applies the comfort temperature
- This ensures Tado doesn't revert to its schedule while you want override active

## Common Use Cases

### Holiday Mode (Always Active)
```yaml
Use Time Window: false
Comfort Temperature: 21°C
```
Turn on when guests arrive, turn off when they leave.

### Weekend Mornings Only
```yaml
Use Time Window: true
Start Time: 07:00:00
End Time: 11:00:00
Comfort Temperature: 22°C
```
Turn on Friday evening, leave it on all weekend!

### Work From Home
```yaml
Use Time Window: true
Start Time: 09:00:00
End Time: 17:00:00
Comfort Temperature: 20.5°C
```
Enable on WFH days, disable on office days.

## Tips

**Note:** The blueprint automatically re-checks and re-applies temperatures every 30 minutes. To customize this interval, you'll need to fork the blueprint and modify the time_pattern trigger.

- **Multiple Zones**: You can select as many Tado zones as you want
- **Multiple Automations**: Create separate automations for different areas with different temperatures
- **Dashboard Control**: Add the override switch to your dashboard for easy access!

## Dashboard Example

Add this card to your Lovelace dashboard:

```yaml
type: entities
title: Tado Override
entities:
  - entity: input_boolean.tado_holiday_override
    name: Holiday Override
  - entity: climate.tado_living_room
  - entity: climate.tado_bedroom
  - entity: climate.tado_office
```

## Need Help?

- Check the [full README](README.md) for detailed documentation
- Review the [example configurations](examples/example_configuration.yaml)
- Open an issue on GitHub if you encounter problems

## Next Steps

Once comfortable with basic operation:
1. Create multiple automations for different scenarios
2. Add automation controls to your dashboard
3. Adjust time windows based on your needs
4. Experiment with time windows for scheduled overrides

Enjoy your flexible Tado heating control! 🔥
