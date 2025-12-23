# Troubleshooting Guide

Common issues and solutions for the Tado Holiday Override Mode blueprint.

## Table of Contents
- [Override Not Working](#override-not-working)
- [Temperatures Reverting](#temperatures-reverting)
- [Time Window Issues](#time-window-issues)
- [Presence Detection Problems](#presence-detection-problems)
- [Blueprint Import Issues](#blueprint-import-issues)
- [Tado Connection Problems](#tado-connection-problems)
- [Getting Help](#getting-help)

---

## Override Not Working

### Symptom
Override switch is ON, but temperatures aren't changing.

### Diagnostic Steps

1. **Check Automation State**
   - Go to **Settings** → **Automations & Scenes**
   - Find your Holiday Override automation
   - Verify it shows as **enabled** (blue toggle)

2. **Check Override Switch State**
   ```
   Settings → Devices & Services → Helpers
   → Find your override switch
   → Verify it shows "On"
   ```

3. **Check Presence Sensor**
   - Verify presence entity shows correct state
   - Common states: "home", "on", "yes", "true"
   - Check entity in **Developer Tools** → **States**

4. **Review Logs**
   ```
   Settings → System → Logs
   → Search for "Tado Holiday Override"
   ```

### Common Causes & Solutions

| Cause | Solution |
|-------|----------|
| Automation disabled | Enable it in Automations list |
| Wrong presence state | Check entity state format (home vs on) |
| Time window restricting | Disable time window or adjust times |
| Climate entities offline | Check Tado integration status |
| Blueprint needs restart | Restart Home Assistant |

### Quick Fix Checklist
- [ ] Automation is enabled
- [ ] Override switch is ON
- [ ] Someone is "home" or presence is "on"
- [ ] Time is within configured window (if enabled)
- [ ] Climate entities are available

---

## Temperatures Reverting

### Symptom
Temperatures are set correctly initially but Tado reverts them to schedule.

### Why This Happens
Tado's internal scheduling can override external temperature commands. The blueprint's periodic check is designed to handle this.

### Solutions

#### 1. Decrease Check Interval
```yaml
Check Interval: 15 minutes  # Instead of 30
```
More frequent checks = less time for Tado to stay at schedule temp.

#### 2. Check Tado Smart Schedule Settings
In the Tado app:
1. Open the room/zone
2. Check if "Smart Schedule" is very aggressive
3. Consider temporarily disabling "Early Start" for the zone

#### 3. Verify HVAC Mode
Ensure the automation is setting the correct HVAC mode:
- Check your climate entities support "heat" mode
- Some Tado setups use "auto" instead

#### 4. Monitor the Automation
Watch the logs during a revert:
```
Settings → System → Logs
→ Filter by "Tado Holiday Override"
→ Look for "Setting X zones to Y°C" messages
```

Should see this message every X minutes while override is active.

### Advanced: Tado Conflict Resolution

Some Tado devices have conflict resolution settings:
1. In Tado app → Settings → Rooms
2. Look for "Manual Control" or "Override" settings
3. Adjust to prefer manual/external commands

---

## Time Window Issues

### Symptom 1: Override not activating during configured window

**Diagnostic:**
```yaml
# Your configuration:
Use Time Window: true
Start Time: 08:00:00
End Time: 22:00:00

# Current time: 15:00 (3 PM)
# Expected: Should be active
# Actual: Not active
```

**Check:**
1. Verify "Use Time Window" is actually `true`
2. Check logs for "outside time window" message
3. Verify Home Assistant system time is correct:
   ```
   Developer Tools → States
   → Search for "sensor.time"
   → Verify it matches your actual time
   ```

**Solution:**
```yaml
# Temporarily disable time window to test
Use Time Window: false

# If that works, the time window logic is the issue
# Check timezone settings in HA configuration.yaml
```

### Symptom 2: Override active outside window

**Cause:** Time window disabled or start/end times swapped

**Check configuration:**
```yaml
# Correct for daytime override:
Start Time: 08:00:00  # Morning
End Time: 22:00:00    # Night

# Correct for overnight override:
Start Time: 22:00:00  # Night
End Time: 08:00:00    # Morning (next day)
```

### Symptom 3: Midnight crossing issues

**Issue:** Windows crossing midnight (e.g., 22:00 to 06:00)

**Current Behavior:**
- HA's time condition handles this automatically
- Should work correctly with proper start/end times

**If Not Working:**
```yaml
# Try adjusting times slightly:
Start Time: 22:00:01
End Time: 05:59:59
```

---

## Presence Detection Problems

### Symptom
Override active even when nobody is home (or vice versa).

### Diagnosis

1. **Check Entity State**
   ```
   Developer Tools → States
   → Find your presence entity
   → Check current state
   ```

2. **Common State Values**
   - Person entities: "home", "not_home", "away"
   - Group entities: "on", "off", "home", "not_home"
   - Binary sensors: "on", "off"
   - Zone entities: "zone.home", "away"

3. **Blueprint Checks For**
   ```yaml
   state: "home"  OR  state: "on"
   ```

### Solutions

#### Wrong State Format
If your presence entity uses different states:

**Option 1: Use a template binary sensor**
```yaml
# In configuration.yaml
binary_sensor:
  - platform: template
    sensors:
      anyone_home:
        friendly_name: "Anyone Home"
        value_template: >-
          {{ is_state('person.john', 'home') or 
             is_state('person.jane', 'home') }}
```

**Option 2: Use a group**
```yaml
# In configuration.yaml
group:
  family:
    name: Family
    entities:
      - person.john
      - person.jane
```

#### Delayed Updates
- Person entities may take time to update
- GPS/wifi tracking has inherent delays

**Workaround:**
Accept that override might run 5-10 minutes longer/shorter than actual presence.

---

## Blueprint Import Issues

### Symptom
Blueprint won't import or shows errors.

### Import URL Issues

**Correct URL:**
```
https://github.com/Bibbleq/HA-Tado-Holiday_Mode/blob/main/blueprints/automation/tado_holiday_override.yaml
```

**Common Mistakes:**
- Missing `/blob/main/`
- Using `/tree/main/` instead
- Using repository root URL

### Manual Import Method

If URL import fails:

1. **Download the file**
   - Go to the [blueprint file](https://github.com/Bibbleq/HA-Tado-Holiday_Mode/blob/main/blueprints/automation/tado_holiday_override.yaml)
   - Click "Raw" button
   - Save as `tado_holiday_override.yaml`

2. **Copy to Home Assistant**
   ```
   config/
   └── blueprints/
       └── automation/
           └── tado_holiday_override/
               └── tado_holiday_override.yaml
   ```

3. **Restart Home Assistant**

### YAML Syntax Errors

If you get YAML errors:
- Don't modify the downloaded file
- Use the exact file from repository
- Check you saved it with `.yaml` extension

---

## Tado Connection Problems

### Symptom
Climate entities show "unavailable" or "unknown".

### Check Tado Integration

1. **Verify Integration**
   ```
   Settings → Devices & Services
   → Find "Tado"
   → Check status (should be green/ok)
   ```

2. **Reload Integration**
   ```
   Settings → Devices & Services → Tado
   → Click three dots menu
   → Reload
   ```

3. **Check Entities**
   ```
   Settings → Devices & Services → Tado
   → Click on device
   → Verify all climate entities are available
   ```

### Authentication Issues

If Tado integration shows errors:
1. Re-enter credentials
2. Check Tado account is active
3. Verify Tado service is online (check tado.com)

### Entity Name Changes

If you renamed Tado entities:
1. Edit the automation
2. Update climate zone selections
3. Save changes

---

## Performance Issues

### Symptom
Home Assistant slow or unresponsive after enabling override.

### Causes & Solutions

#### Too Many Zones
```yaml
# If controlling >10 zones:
# Split into multiple automations
# Group zones by area/floor
```

#### Interval Too Short
```yaml
# Minimum recommended:
Check Interval: 10 minutes

# For many zones:
Check Interval: 30 minutes
```

#### Tado API Rate Limiting
- Tado may throttle too many requests
- Increase interval between checks
- Reduce number of zones

### Monitor System Load
```
Settings → System → System Health
→ Check CPU/Memory usage
```

---

## Debugging Tips

### Enable Debug Logging

Add to `configuration.yaml`:
```yaml
logger:
  default: info
  logs:
    homeassistant.components.automation: debug
```

Then restart HA and check logs.

### Watch Automation in Real-Time

```
Developer Tools → Events
→ Listen to event: "automation_triggered"
→ Turn override on/off
→ Watch for your automation
```

### Test Individual Components

1. **Test presence entity:**
   ```
   Developer Tools → States
   → Find presence entity
   → Manually set state
   ```

2. **Test climate control:**
   ```
   Developer Tools → Services
   → Service: climate.set_temperature
   → Entity: climate.tado_living_room
   → Temperature: 21
   → Call Service
   ```

3. **Test time window:**
   ```
   Temporarily set wide window (00:00 - 23:59)
   Verify override works
   Narrow window incrementally
   ```

---

## Common Error Messages

### "could not determine a constructor for the tag '!input'"
- **Meaning:** You're trying to validate YAML outside of Home Assistant
- **Solution:** This is normal - the file is valid for HA, ignore this error

### "Entity not found: input_boolean.xxx"
- **Meaning:** The override switch doesn't exist
- **Solution:** Create the helper first (see Quick Start Guide)

### "Platform climate does not generate unique IDs"
- **Meaning:** Tado integration issue, not blueprint issue
- **Solution:** Restart Tado integration or reinstall it

### "Error rendering data template"
- **Meaning:** Template syntax issue in automation
- **Solution:** Re-import blueprint (you may have an old version)

---

## Still Having Issues?

### Gather Information

Before asking for help, collect:
1. **Home Assistant version:**
   ```
   Settings → About → Version
   ```

2. **Blueprint version/date downloaded:**
   ```
   Check when you imported/downloaded
   ```

3. **Relevant configuration:**
   ```yaml
   # Sanitized version (remove personal details):
   Override Switch: input_boolean.xxx
   People Presence: person.xxx
   Tado Zones: climate.tado_xxx
   Check Interval: 30
   Use Time Window: true/false
   ```

4. **Error messages:**
   ```
   Settings → System → Logs
   → Copy any "Tado Holiday Override" errors
   ```

5. **What you've tried:**
   - List troubleshooting steps already attempted

### Getting Help

1. **GitHub Issues** (Recommended):
   - Go to [Issues](https://github.com/Bibbleq/HA-Tado-Holiday_Mode/issues)
   - Search for similar problems first
   - Create new issue with information above

2. **Home Assistant Community**:
   - [Home Assistant Community Forum](https://community.home-assistant.io)
   - Tag with "tado" and "blueprint"

3. **Reddit**:
   - r/homeassistant
   - Provide full context and troubleshooting steps tried

### Provide Good Bug Reports

**Good:**
```
Title: Override not activating during time window

Description:
- HA Version: 2024.1.0
- Blueprint imported via URL on 2024-01-15
- Override switch: input_boolean.tado_holiday_override (ON)
- Presence: person.john (state: home)
- Time window: 08:00-22:00 (enabled)
- Current time: 15:00
- Expected: Temps set to 21°C
- Actual: No change to temps
- Logs show: "Conditions not met for applying override"
- Tried: Disabling time window (works), re-enabling (fails)
```

**Bad:**
```
Title: Not working

Description:
It doesn't work. Please help.
```

---

## Preventive Maintenance

### Regular Checks (Monthly)

- [ ] Verify automation still enabled
- [ ] Check Tado integration status
- [ ] Review logs for errors
- [ ] Test override activation/deactivation
- [ ] Verify presence detection accuracy

### After Home Assistant Updates

- [ ] Test override functionality
- [ ] Check for any new errors in logs
- [ ] Verify Tado integration still works
- [ ] Re-import blueprint if major changes

### After Tado App Updates

- [ ] Test climate entity control
- [ ] Verify preset modes still work
- [ ] Check for any new Tado features to leverage

---

## Known Limitations

1. **Not Instantaneous**
   - Periodic checks mean temps may revert briefly
   - Solution: Decrease interval

2. **Requires Internet**
   - Tado cloud API needed
   - Local control not available

3. **No Schedule Editing**
   - Blueprint doesn't modify Tado schedules
   - Only overlays temporary settings

4. **Single Temperature Per Automation**
   - All zones get same temperature
   - Solution: Create multiple automations

---

## FAQ

**Q: Can I use this for cooling?**
A: Modify the blueprint and change `hvac_mode: "heat"` to `hvac_mode: "cool"`

**Q: Works with Tado AC?**
A: Yes, as long as climate entities are available

**Q: Multiple temperature zones?**
A: Create separate automations for each temperature requirement

**Q: Battery impact on phone (presence)?**
A: Minimal - automation runs server-side, not on phone

**Q: Can I use with other climate platforms?**
A: Yes, but you may need to adjust preset modes

---

Remember: Most issues are configuration-related, not blueprint bugs. Work through this guide systematically before reporting issues.
