# Heat Protection Blueprint for Home Assistant

Automatically close covers (roller shutters, blinds, awnings) in the morning when a hot day is forecast, and reopen them in the evening once it's dark enough. Respects window state — covers only close when the corresponding window is shut.

[![Open your Home Assistant instance and show the blueprint import dialog with this blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fs256%2Fhomeassistant-hitzeschutz%2Fblob%2Fmain%2Fheat_protection_blueprint.yaml)

## Features

- **Forecast-based**: Closes covers proactively based on predicted high temperature, not reactively when it's already hot.
- **Window-aware**: Only closes covers where the corresponding window is shut (index-matched pairing).
- **Elevation-based opening**: Opens in the evening based on sun elevation angle — works correctly year-round regardless of season.
- **HA restart recovery**: Re-evaluates and closes covers after a Home Assistant restart mid-day.
- **Respects manual override**: If you manually open a cover (e.g. for ventilation), it won't be automatically closed again.
- **Notifications**: Alerts you when a window is open (cover skipped) or a sensor is unavailable (connectivity issue).
- **Multi-cover support**: Handles any number of cover/window-sensor pairs.

## Architecture

```
+-------------------+     +------------------------------+
| Template Sensor   |     |        BLUEPRINT             |
| Forecast High     |     |  (one automation, 3 paths)   |
| Temperature       |     +------------------------------+
| (updated hourly)  |              |
+-------------------+              |
        |                          v
        |         +--------------------------------+
        +-------->| CLOSE                          |
                  | Trigger: Sunrise OR            |
                  |          window closed         |
                  | Check: Forecast > threshold    |
                  |        Window shut? (by index) |
                  +--------------------------------+
                               |
                               | heat_protection_active = ON
                               v
                  +--------------------------------+
                  | OPEN                           |
                  | Trigger: Elevation < threshold |
                  |          (for: 10 min)         |
                  | Condition: after 18:00         |
                  +--------------------------------+
                               |
                               v
                  +--------------------------------+
                  | RECOVERY (HA restart)          |
                  | Waits 2 min, re-evaluates,     |
                  | closes if needed               |
                  +--------------------------------+
```

## Prerequisites

- Home Assistant 2024.1+
- Cover integration (e.g. Velux KLF 200, Somfy, Shelly, any Z-Wave/Zigbee cover)
- Weather integration with forecast support (e.g. Met.no, OpenWeatherMap, AccuWeather)
- Binary sensors for window open/close state

## Installation

### Step 1: Create the Forecast High Temperature Sensor

You need a template sensor that provides today's forecast high temperature. Add this to your `configuration.yaml` (or in Settings > Devices & Services > Helpers > Template):

**Trigger-based template sensor:**

```yaml
template:
  - trigger:
      - trigger: time_pattern
        minutes: "/30"
      - trigger: homeassistant
        event: start
    action:
      - action: weather.get_forecasts
        data:
          type: hourly
        target:
          entity_id: weather.YOUR_WEATHER_ENTITY
        response_variable: hourly_forecast
      - action: weather.get_forecasts
        data:
          type: daily
        target:
          entity_id: weather.YOUR_WEATHER_ENTITY
        response_variable: daily_forecast
    sensor:
      - name: "Forecast High Temperature"
        unique_id: forecast_high_temperature
        device_class: temperature
        unit_of_measurement: "°C"
        icon: mdi:thermometer-chevron-up
        state: >
          {% set forecasts = hourly_forecast['weather.YOUR_WEATHER_ENTITY']['forecast'][:12] %}
          {% set temps = forecasts | map(attribute='temperature') | list %}
          {{ temps | max if temps | length > 0 else 'unavailable' }}
        attributes:
          tomorrow_high: >
            {% set forecast = daily_forecast['weather.YOUR_WEATHER_ENTITY']['forecast'] %}
            {{ forecast[1].temperature if forecast | length > 1 else 'unavailable' }}
          tomorrow_condition: >
            {% set forecast = daily_forecast['weather.YOUR_WEATHER_ENTITY']['forecast'] %}
            {{ forecast[1].condition if forecast | length > 1 else 'unknown' }}
          updated_at: "{{ now().isoformat() }}"
```

> Replace `weather.YOUR_WEATHER_ENTITY` with your actual weather entity (e.g. `weather.home`, `weather.openweathermap`).

### Step 2: Create an Input Boolean Helper

Settings > Devices & Services > Helpers > Create Helper > Toggle

- Name: `Heat Protection Active`
- Resulting entity: `input_boolean.heat_protection_active`

This tracks whether heat protection has been activated today.

### Step 3: Import the Blueprint

Click the badge at the top of this README, or manually:

1. Settings > Automations & Scenes > Blueprints > Import Blueprint
2. Paste URL: `https://github.com/s256/homeassistant-hitzeschutz/blob/main/heat_protection_blueprint.yaml`

Or copy `heat_protection_blueprint.yaml` to:
```
config/blueprints/automation/heat_protection/heat_protection.yaml
```

### Step 4: Create an Automation from the Blueprint

Settings > Automations & Scenes > Create Automation > Use Blueprint > "Heat Protection"

| Field | Description |
|-------|-------------|
| Covers | Select all covers to control (any number) |
| Window Sensors | Corresponding window sensors (same order!) |
| Temperature Sensor | Your `sensor.forecast_high_temperature` |
| Temperature Threshold | Forecast temp above which covers close (default: 27°C) |
| Darkness Elevation | Sun elevation at which covers reopen in the evening (default: -5°) |
| Notification Target | Notify service target, e.g. `mobile_app_yourphone` (optional, leave empty to disable) |
| Day Status Toggle | Your `input_boolean.heat_protection_active` |

> **Important**: Covers and window sensors must be in the same order!
> Cover 1 pairs with Window Sensor 1, Cover 2 with Window Sensor 2, etc.

### Step 5: Verify

After saving:
1. Check `sensor.forecast_high_temperature` shows a numeric value
2. The automation is enabled
3. Test with Developer Tools > Template: `{{ states('sensor.forecast_high_temperature') }}`

## Configuration

All settings are configured directly in the blueprint inputs — no extra helpers needed (except the one input_boolean for day status).

| Setting | Default | Description |
|---------|---------|-------------|
| Temperature Threshold | 27°C | Forecast high above which covers close |
| Darkness Elevation | -5° | Sun elevation below which covers reopen |

**Recommended elevation values:**

| Value | Meaning |
|-------|---------|
| -4° | Light twilight — fine for most projectors |
| -5° | Good compromise (default) |
| -6° | End of civil twilight — definitely dark |
| -8° | Very dark — for sensitive projectors |

> At ~51°N latitude, -5° elevation corresponds to approximately 22:30–23:00 in June.

## Edge Cases & Handling

### Critical

| Problem | Solution |
|---------|----------|
| Window sensor reports `unavailable` | Cover skipped + notification sent (Zigbee/connectivity issue) |
| Forecast sensor unavailable | Condition check prevents automation from running |
| HA restart mid-day | Recovery path: wait 2 min, re-evaluate, close if needed |

### Medium

| Problem | Solution |
|---------|----------|
| Window closed later in the day | Trigger on window state change → catches up |
| Cover manually opened (e.g. ventilation) | Will NOT be automatically closed again |
| Forecast changes (cooler) | Once closed, stays closed until evening |

### Low

| Problem | Solution |
|---------|----------|
| Transition seasons (spring/fall) | Adjust threshold in blueprint settings |
| Forecast data delayed at startup | 2 min delay in recovery path handles this |

## Design Decisions

1. **Per-window logic via index mapping**: Cover[0] belongs to Window[0]. Supports any number of pairs without hardcoding.

2. **No re-close after manual open**: User intent is respected.

3. **Opening is always safe**: In the evening, all covers open — no window check needed.

4. **Elevation instead of time offset**: `sunset + 2h` would be too late in December, too early in June. Elevation is season-independent.

5. **Forecast instead of current temperature**: Proactive morning close, not reactive when it's already hot.

6. **Single blueprint, no package**: No separate input_number helpers. Configuration lives in the blueprint. Only external helper: one input_boolean for day status.

## Extension Ideas

- **Partial close**: Replace `cover.close_cover` with `cover.set_cover_position` in the blueprint for partial shading.

- **Time window**: Add a `condition: time` (e.g. not before 06:00) for bedroom covers.

- **Tomorrow preview notification**: The sensor has `tomorrow_high` as an attribute. Build a notification:
  ```yaml
  "Tomorrow's forecast high: {{ state_attr('sensor.forecast_high_temperature', 'tomorrow_high') }}°C"
  ```

## License

MIT — see [LICENSE](LICENSE).
