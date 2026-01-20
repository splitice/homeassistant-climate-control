# Implementation Summary: v4 Evaporative Cooler Control

## Overview
Implemented v4 of the evaporative cooler control system with improved performance, removing indoor humidity estimation and adding pad drying progress tracking with intelligent dual-mode temperature control.

## Changes Made

### 1. New Files Created

#### `evap_set_evap_target.script.yml`
- Core helper script for all mode and fan speed changes
- Enforces safety constraints:
  - Maximum 2-step fan speed changes per cycle
  - Gradual transition when turning off (via fan_only first)
  - Applies min/max fan speed limits
- Used by all other scripts for consistent behavior
- Includes retry logic for reliable state changes

#### `evap_pad_drying.script.yml`
- Manages evaporative pad drying cycle
- Key features:
  - Tracks drying progress as percentage (0-100%)
  - Configurable drying times based on fan speed
  - Linear interpolation between speed 1 and speed 10 times
  - Gradually lowers fan speed as pads dry
  - Automatically turns off when 100% dry
  - Lower fan speed when indoor temp below setpoint or >80% dried

#### `evap_maintain_temperature.script.yml` (v4)
- Gentle temperature maintenance mode
- Key features:
  - Uses temperature change rate when available (°C/min)
  - Maximum ±1 fan speed change per cycle
  - Compares feels-like temps at -1, current, +1 fan speeds
  - Selects speed closest to setpoint feels-like temperature
  - Switches to fan_only if below setpoint
- Uses inline wet bulb calculation (Stull approximation)
- Calculates feels-like temperature with fan airflow effect

#### `evap_get_to_target_temperature.script.yml`
- Aggressive cooling to reach target temperature
- Key features:
  - Calculates target minutes based on temp difference
  - Tests each fan speed (1-10) to estimate time to target
  - Selects speed that reaches target closest to desired time
  - Maximum ±2 fan speed change per cycle (enforced by set_evap_target)
  - Always operates in cool mode
- Uses feels-like temperature with cooling effect estimation

### 2. Main Automation File

#### `evap-control.automation.yml` (v4)
Complete rewrite with major improvements:

**New State Variables:**
- `pad_drying_enabled`: Tracks if pad drying is active
- `temperature_change_per_minute`: Temperature trend for predictive control
- `pad_drying_previous`: Unix timestamp for drying duration calculations
- `alg_state`: Current algorithm state for logging

**Control Flow:**
1. Check auto control state and restore snapshot if off
2. Take snapshot of evap state on startup
3. Main control loop:
   - Calculate wet bulb temperature inline
   - Calculate indoor feels-like temperature accounting for:
     - Fan speed airflow effect (0.1-1.5 m/s)
     - Cooling mode effect (4°C max drop at speed 10)
     - Fan_only mode effect (2°C max drop at speed 10)
     - Humidity effect via apparent temperature
   - Determine if needs cooling, at target, or should maintain
   - Route to appropriate script:
     - `evap_get_to_target_temperature` if diff > maintain_threshold
     - `evap_maintain_temperature` if diff ≤ maintain_threshold
     - `evap_pad_drying` if cooling not needed
   - Log state changes with algorithm name
   - Wait for changes (60s if changed, 30s if not)
   - Calculate temperature change rate for next cycle

**Key Improvements:**
- Clear separation of get-to-target vs maintain modes
- Pad drying progress tracking replaces time-based approach
- Feels-like temperature calculated inline (no sensor needed)
- Temperature change rate enables predictive control
- Logging includes algorithm state for debugging

### 3. Archived v3 Files

Renamed with `.v3.yml` extension for reference:
- `evap-control.automation.v3.yml` (old main automation)
- `evap_maintain_temperature.script.v3.yml` (old maintain script)
- `evap_apply_mode_and_fan_with_twb.script.v3.yml` (old aggressive cooling)
- `evap_ramp_fan_in_fan_only.script.v3.yml` (old ramping logic)

### 4. Updated Documentation

#### `README.md`
Complete rewrite covering:
- v4 feature highlights
- Three control modes (get to target, maintain, pad drying)
- Design rationale and improvements
- Required entities with configuration examples
- Configuration variables and tuning parameters
- Script descriptions
- v4 improvements over v3
- Migration guide from v3
- Installation and troubleshooting

## v4 Key Features

### Removed Complexity
1. **No Indoor Humidity Estimation**: Removed unreliable estimation logic
2. **No Separate Wet Bulb Sensor**: Calculated inline using Stull approximation
3. **No Time-Based Pad Drying**: Replaced with progress percentage

### New Capabilities
1. **Pad Drying Progress**: Visual 0-100% tracking with configurable times
2. **Feels-Like Temperature**: Accounts for humidity and fan airflow
3. **Temperature Change Rate**: Predictive control using temperature trend
4. **Dual-Mode Operation**: Clear get-to-target vs maintain separation

### Enhanced Safety
1. **Maximum 2-Step Changes**: Prevents mechanical stress and noise
2. **Gradual Shutdown**: Always transitions through fan_only before off
3. **Constraint Enforcement**: Min/max fan speed limits applied consistently

## Control Algorithm

### Get to Target Mode (diff > 1°C)
```
1. Calculate target_minutes = max(3, 3 - (diff × 0.5))
2. For each fan speed 1-10:
   - Calculate feels-like temp with that speed
   - Estimate minutes to reach target
3. Select speed closest to target_minutes
4. Cap change to ±2 steps from current speed
```

### Maintain Temperature Mode (diff ≤ 1°C)
```
If temperature_change_per_minute available:
  - If diff > 0.6°C: increase fan by 1
  - If diff < 0°C: decrease fan by 1
  - If cooling and diff < 0: decrease fan by 1
  - If warming and diff > 0: increase fan by 1
Else:
  - Test fan speeds: current-1, current, current+1
  - Calculate feels-like temp at each speed
  - Select speed with feels-like closest to setpoint
```

### Pad Drying Mode
```
1. Reset progress to 0% when starting
2. Ensure in fan_only mode
3. Calculate progress: elapsed_time / expected_time × 100%
   - expected_time interpolated from speed 1 and speed 10 times
4. Lower fan speed if:
   - Indoor temp < setpoint (don't make it colder), OR
   - Progress ≥ 80% (nearly done)
5. Turn off when progress ≥ 100%
```

## Technical Details

### Feels-Like Temperature Calculation
```
fan_fraction = (fan_speed - 1) / 9
T_effective = T_indoor - (cooling_effect × fan_fraction)
  - cooling_effect = 4°C in cool mode, 2°C in fan_only mode
air_velocity = 0.10 + (fan_speed - 1) × (1.50 - 0.10) / 9
vapor_pressure = (RH / 100) × saturation_vapor_pressure(T_effective)
apparent_temp = T_effective + 0.33 × vapor_pressure - 0.70 × air_velocity - 4.0
```

### Wet Bulb Temperature (Stull Approximation)
```
Twb = T × atan(0.151977 × √(RH + 8.313659))
      + atan(T + RH)
      - atan(RH - 1.676331)
      + 0.00391838 × RH^1.5 × atan(0.023101 × RH)
      - 4.686035
```

## Validation

### YAML Syntax
All files validated with Python yaml.safe_load():
- ✓ evap_set_evap_target.script.yml
- ✓ evap_pad_drying.script.yml  
- ✓ evap_maintain_temperature.script.yml
- ✓ evap_get_to_target_temperature.script.yml
- ✓ evap-control.automation.yml

### Logic Flow
Verified correct behavior for:
- Mode transitions (off → fan_only → cool)
- Fan speed constraints (max ±2 steps)
- Pad drying progress calculation
- Temperature change rate tracking
- Feels-like temperature with fan airflow
- Script routing based on temperature difference

## Requirements Checklist

Based on IMPLEMENTATION.v4.md specification:

- [x] **Main Control Loop**: Persistent automation with state tracking
- [x] **Maintain Temperature Script**: Minimal changes, temperature-based selection
- [x] **Get to Target Script**: Aggressive cooling with time-based speed selection
- [x] **Pad Drying Script**: Progress tracking with configurable times
- [x] **Set Evap Target Script**: Safe mode/speed changes with constraints
- [x] **Wet Bulb Calculation**: Inline Stull approximation
- [x] **Feels-Like Temperature**: Accounts for humidity and fan airflow
- [x] **Temperature Change Rate**: Tracks and uses for predictive control
- [x] **Pad Drying Progress**: 0-100% tracking with interpolated times
- [x] **Fan Speed Constraints**: Max ±2 steps, min/max limits
- [x] **Dual-Mode Operation**: Clear separation of get-to-target and maintain

## Migration Notes

### For Existing v3 Users
1. Create `input_number.pad_drying_progress` entity
2. Replace automation and script files
3. Optional: Remove old sensors (viewbank_wet_bulb_temperature, home_feels_like)
4. Test and tune parameters as needed

### Breaking Changes
- Requires new `pad_drying_entity` input_number
- Different script names (old scripts archived as .v3.yml)
- Different control logic (may behave differently initially)

### Backward Compatibility
- Same automation entity name
- Same required climate entity
- Same input entities (temperature, control boolean)
- Old v3 files preserved for reference

## Future Enhancements

Possible improvements:
- Machine learning for optimal fan speed selection
- Weather forecast integration for proactive cooling
- Adaptive maintain threshold based on conditions
- Energy usage tracking and optimization
- Configurable feels-like temperature parameters
