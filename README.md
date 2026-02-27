# Home Assistant Evaporative Cooler Automation (v4)

This automation controls an evaporative cooler to maintain a target temperature with improved performance, pad drying tracking, and intelligent mode switching between aggressive cooling and temperature maintenance.

## Key Features (v4)

- **Dual-Mode Temperature Control**: Automatically switches between aggressive "get to target" cooling and gentle "maintain temperature" modes
- **Pad Drying Management**: Tracks pad drying progress (0-100%) with configurable drying times based on fan speed
- **Feels-Like Temperature**: Uses apparent temperature calculation accounting for humidity and fan speed airflow effects
- **Adaptive Fan Control**: Fan speed adjusts based on temperature difference, wet bulb conditions, and temperature change rate
- **Safe State Transitions**: Maximum 2-step fan speed changes with intelligent mode switching
- **Snapshot/Restore**: Safely exits automation by restoring previous cooler state

## Control Modes

### Get to Target Mode
Used when temperature is significantly above setpoint (>1°C difference):
- Aggressively cools to reach target temperature
- Selects fan speed based on estimated time to reach target
- Can increase fan speed by up to 2 steps per cycle
- Uses feels-like temperature with wet bulb calculations

### Maintain Temperature Mode
Used when temperature is within 1°C of setpoint:
- Makes minimal fan speed adjustments (±1 step maximum)
- Uses temperature change rate when available for smoother control
- Compares feels-like temperature at adjacent fan speeds to find optimal setting
- Prevents frequent cycling with intelligent thresholds
- **Dehumidify Mode**: When in cool mode and indoor temperature is 3°C or more below setpoint while feels-like temperature is comfortable, outdoor conditions are favorable, and indoor humidity is significantly higher than outdoor (>10% difference and >50% absolute), switches to fan_only mode with reduced fan speed to decrease humidity without overcooling
- **Outdoor-Hotter Switching**: When outdoor temperature exceeds indoor temperature and cool mode is allowed, automatically switches to cool mode instead of increasing fan speed in fan_only mode — blowing hotter outdoor air indoors would warm the house, whereas evaporative cooling reduces supply air temperature below the outdoor dry-bulb temperature

### Pad Drying Mode
Activated when cooling is no longer needed:
- Tracks drying progress (0-100%) based on fan speed and elapsed time
- Gradually lowers fan speed as pads dry
- Configurable drying times (default: 10 minutes at speed 1, 5 minutes at speed 10)
- Automatically turns off when 100% dry

## Design Rationale

- **Dual-Mode Operation**: Separate scripts for aggressive cooling (get_to_target) and maintenance provide optimal performance in each scenario
- **Feels-Like Temperature**: More accurate comfort measurement by accounting for humidity and fan airflow effects (2-4°C cooling effect based on mode and fan speed)
- **Pad Drying Progress Tracking**: Percentage-based tracking (0-100%) ensures pads are fully dried before shutdown, preventing mold and extending pad life
- **Wet Bulb Temperature**: Calculated inline using Stull approximation for evaporative cooling efficiency
- **Temperature Change Rate**: When available, uses temperature trend (°C/min) for predictive fan speed adjustments
- **Safe State Transitions**: Maximum 2-step fan speed changes prevent mechanical stress and excessive noise
- **Snapshot/Restore**: Preserves user settings and allows safe exit from automation
- **No Indoor Humidity Estimation**: v4 removes complex indoor humidity estimation, relying on feels-like temperature calculations instead

## Required Entities

The automation requires the following Home Assistant entities:

### Core Entities
- `select.roofevap1_e90248_evap_mode` - The evaporative cooler mode select entity (options: "off", "fan_only", "cool")
- `select.roofevap1_e90248_evap_fan_speed` - The evaporative cooler fan speed select entity (options: "1" through "10")
- `input_number.target_temperature` - Target temperature setpoint
- `input_boolean.automatic_control` - Toggle for automatic control
- `input_number.pad_drying_progress` - Tracks pad drying progress (0-100%)

### Temperature & Humidity Sensors
- `sensor.home_temperature` - Indoor temperature sensor
- `sensor.gw3000c_outdoor_temperature` - Outdoor dry-bulb temperature sensor (with fallback to `sensor.viewbank_temp`)
- `sensor.gw3000c_humidity` - Outdoor relative humidity sensor (with fallback to `sensor.viewbank_humidity`)
- `sensor.climate_indoor_humidity` - Indoor relative humidity sensor (with fallback to `sensor.viewbank_humidity`)

### Optional Entities
- `input_number.min_fan_speed` - Minimum fan speed constraint (1-10)
- `input_number.max_fan_speed` - Maximum fan speed constraint (1-10)

**Note**: The automation uses the following sensors with automatic fallback to viewbank sensors if unavailable:
- `sensor.gw3000c_outdoor_temperature` (fallback: `sensor.viewbank_temp`) for outdoor temperature
- `sensor.gw3000c_humidity` (fallback: `sensor.viewbank_humidity`) for outdoor humidity
- `sensor.climate_indoor_humidity` (fallback: `sensor.viewbank_humidity`) for indoor humidity

The automation calculates wet bulb temperature inline using the Stull approximation. The evaporative cooler entities are ESPHome-based select entities that control mode and fan speed separately.

### Creating Required Input Entities

Add these to your Home Assistant configuration:

```yaml
input_number:
  target_temperature:
    name: Target Temperature
    min: 18
    max: 28
    step: 0.5
    unit_of_measurement: "°C"
    icon: mdi:thermometer
  
  pad_drying_progress:
    name: Pad Drying Progress
    min: 0
    max: 100
    step: 0.1
    unit_of_measurement: "%"
    icon: mdi:water-percent

input_boolean:
  automatic_control:
    name: Automatic Evap Control
    icon: mdi:fan-auto
```

## Configuration

The automation can be customized through variables defined at the top of `evap-control.automation.yml`:

```yaml
variables:
  # Core entities (required)
  evap_entity: select.roofevap1_e90248_evap_mode
  fan_speed_entity: select.roofevap1_e90248_evap_fan_speed
  setpoint_entity: input_number.target_temperature
  auto_entity: input_boolean.automatic_control
  
  # Primary sensor entities with fallback support
  outdoor_temp_entity_primary: sensor.gw3000c_outdoor_temperature
  outdoor_temp_entity_fallback: sensor.viewbank_temp
  outdoor_humidity_entity_primary: sensor.gw3000c_humidity
  outdoor_humidity_entity_fallback: sensor.viewbank_humidity
  indoor_temperature_entity: sensor.home_temperature
  indoor_humidity_entity_primary: sensor.climate_indoor_humidity
  indoor_humidity_entity_fallback: sensor.viewbank_humidity
  
  pad_drying_entity: input_number.pad_drying_progress
  
  # Optional constraints
  max_fan_speed_entity: null  # or input_number.max_fan_speed
  min_fan_speed_entity: null  # or input_number.min_fan_speed
  
  # Tuning parameters
  maintain_threshold: 1.0  # °C - switch to maintain mode within this range
  pad_dry_time_speed1: 10  # minutes - drying time at fan speed 1
  pad_dry_time_speed10: 5  # minutes - drying time at fan speed 10
  deadband_cooling_upper: 0.3  # °C - start cooling above this
  deadband_cooling_lower: 0.0  # °C - stop cooling below this
```

### Tuning Parameters

- **maintain_threshold**: Temperature difference (°C) to switch from aggressive to maintain mode. Default: 1.0°C
- **pad_dry_time_speed1/10**: Pad drying duration at minimum and maximum fan speeds. Times for intermediate speeds are interpolated linearly
- **deadband_cooling_upper/lower**: Temperature deadband for cooling decisions. Creates hysteresis to prevent rapid cycling

## Scripts

The automation uses four scripts:

1. **evap_set_evap_target.script.yml**: Core helper script that safely changes HVAC mode and fan speed with constraints
2. **evap_get_to_target_temperature.script.yml**: Aggressive cooling to reach target temperature quickly
3. **evap_maintain_temperature.script.yml**: Gentle temperature maintenance with minimal fan speed changes  
4. **evap_pad_drying.script.yml**: Manages pad drying cycle with progress tracking

All scripts must be added to your Home Assistant scripts configuration.

## v4 Improvements

Version 4 brings significant improvements over v3:

### Removed Features
- **Indoor Humidity Estimation**: Removed complex and unreliable indoor humidity estimation logic
- **Separate Wet Bulb Sensor**: No longer requires pre-configured wet bulb temperature sensor (calculated inline)
- **Time-Based Pad Drying**: Replaced fixed 5-10 minute timers with percentage-based progress tracking

### New Features
- **Pad Drying Progress Entity**: Visual feedback on pad drying status (0-100%)
- **Feels-Like Temperature**: Integrated apparent temperature calculation accounting for fan airflow
- **Temperature Change Rate**: Uses temperature trend for predictive fan speed adjustments
- **Consistent Fan Speed Goal**: Algorithm prioritizes stable fan speeds over rapid adjustments

### Improved Algorithms
- **Better Mode Switching**: Clear separation between get-to-target and maintain modes
- **Smarter Fan Speed Selection**: Evaluates multiple fan speeds to find optimal setting
- **Enhanced Safety Constraints**: Maximum 2-step fan speed changes prevent mechanical stress

## Migration from v3

If upgrading from v3:

1. **Add new required entity**: Create `input_number.pad_drying_progress` (see configuration above)
2. **Update automation**: Replace `evap-control.automation.yml` with v4 version
3. **Update scripts**: Replace all script files with v4 versions
4. **Optional sensors**: The `sensor.viewbank_wet_bulb_temperature` and `sensor.home_feels_like` are no longer required (calculations done inline)
5. **Test thoroughly**: Monitor behavior for the first few cooling cycles

Old v3 files are preserved with `.v3.yml` extension for reference.

## Installation

1. Copy all `.yml` files to your Home Assistant configuration directory
2. Add input entities to `configuration.yaml` (see Required Entities section)
3. Update entity names in automation variables to match your setup
4. Reload automations and scripts in Home Assistant
5. Enable automation by turning on `input_boolean.automatic_control`

## Troubleshooting

- **Automation not starting**: Check that all required entities exist and are available
- **Rapid fan speed changes**: Increase `maintain_threshold` to widen the maintain mode range
- **Pads not drying completely**: Increase `pad_dry_time_speed1` and `pad_dry_time_speed10` values
- **Temperature overshoots target**: Adjust `deadband_cooling_upper` to start cooling earlier

## Contributing

See `IMPLEMENTATION.v4.md` for detailed specification and architecture documentation.

