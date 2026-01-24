# System Architecture and Flow Diagrams (v4)

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Home Assistant                           │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │         Evaporative Cooler Automation (v4)           │ │
│  │                                                       │ │
│  │  ┌────────────┐         ┌──────────────┐           │ │
│  │  │  Sensors   │────────>│ Control Loop │           │ │
│  │  │            │         │              │           │ │
│  │  │ • Home Temp│         │ • Monitor    │           │ │
│  │  │ • Outdoor  │         │ • Decide     │           │ │
│  │  │ • Humidity │         │ • Execute    │           │ │
│  │  │            │         │              │           │ │
│  │  └────────────┘         └──────┬───────┘           │ │
│  │                                 │                    │ │
│  │                    ┌────────────┴────────────┐      │ │
│  │                    │                         │      │ │
│  │              ┌─────▼──────┐           ┌─────▼─────┐│ │
│  │              │Get to      │           │  Maintain ││ │
│  │              │Target      │           │   Temp    ││ │
│  │              │Script      │           │  Script   ││ │
│  │              └─────┬──────┘           └─────┬─────┘│ │
│  │                    │                         │      │ │
│  │                    └────────┬────────────────┘      │ │
│  │                             │                       │ │
│  │                      ┌──────▼──────┐               │ │
│  │                      │ Pad Drying  │               │ │
│  │                      │   Script    │               │ │
│  │                      └──────┬──────┘               │ │
│  │                             │                       │ │
│  │                      ┌──────▼──────┐               │ │
│  │                      │Set Evap     │               │ │
│  │                      │Target       │               │ │
│  │                      │Script       │               │ │
│  │                      └──────┬──────┘               │ │
│  └─────────────────────────────┼──────────────────────┘ │
│                                 │                        │
│                      ┌──────────▼──────────┐            │
│                      │  climate.evap       │            │
│                      │  (HVAC Device)      │            │
│                      └─────────────────────┘            │
└─────────────────────────────────────────────────────────┘
```

## State Machine Diagram (v4)

```
                     ┌─────────────────────┐
                     │                     │
                     │       OFF           │
                     │                     │
                     └──────────┬──────────┘
                                │
                   needs_cooling│
                   (temp not below maintain_threshold)
                                │
                     ┌──────────▼──────────┐
                     │                     │
                     │ GET TO TARGET       │
                     │ (Aggressive Cool)   │
                     │                     │
                     └──────────┬──────────┘
                                │
             diff <= maintain_threshold (1°C)
                                │
                     ┌──────────▼──────────┐
        ┌────────────┤                     │
        │            │   MAINTAIN TEMP     │
        │            │   (Fine Control)    │
        │            │                     │
        │            └──────────┬──────────┘
        │                       │
        │                       │ diff exceeds
        │     needs_cooling     │ maintain_threshold
        │        again          │
        │                       │
        └───────────────────────┘
                                │
             diff < maintain_threshold
             (no longer needs cooling)
                                │
                     ┌──────────▼──────────┐
                     │                     │
                     │     PAD DRYING      │
                     │    (fan_only)       │
                     │   Progress: 0-100%  │
                     │                     │
                     └──────────┬──────────┘
                                │
                   progress = 100%
                                │
                     ┌──────────▼──────────┐
                     │                     │
                     │       OFF           │
                     │                     │
                     └─────────────────────┘
```

## v4 Control Logic Flow

### Temperature Difference Thresholds

```
Temperature Difference (diff = indoor_feels_like - setpoint)

needs_cooling = not (diff < maintain_threshold)
should_maintain = abs(diff) <= maintain_threshold
at_target = (diff <= deadband_cooling_upper and diff >= deadband_cooling_lower)

Default values:
  maintain_threshold: 1.0°C
  deadband_cooling_upper: 0.3°C
  deadband_cooling_lower: 0.0°C
```

### Main Control Decision Tree

```
┌─────────────────────────────────────┐
│  Calculate feels-like temperature   │
│  diff = feels_like - setpoint       │
└──────────────┬──────────────────────┘
               │
    ┌──────────▼─────────────┐
    │ should_maintain OR     │
    │ needs_cooling?         │
    └──┬─────────────────┬───┘
       │ YES             │ NO
       ▼                 ▼
┌──────────────┐   ┌─────────────────┐
│ COOLING MODE │   │  PAD DRYING     │
└──────┬───────┘   │  or OFF         │
       │           └─────────────────┘
       ▼
┌──────────────────────┐
│ should_maintain?     │
│ (diff <= 1.0°C)      │
└──┬────────────────┬──┘
   │ YES            │ NO
   ▼                ▼
┌────────────┐  ┌────────────────┐
│ MAINTAIN   │  │ GET TO TARGET  │
│ MODE       │  │ MODE           │
└────────────┘  └────────────────┘
```

## Script Interactions

```
Main Automation Loop
         │
         ├──> evap_get_to_target_temperature.script.yml
         │    (Aggressive cooling to reach target)
         │    │
         │    └──> evap_set_evap_target.script.yml
         │
         ├──> evap_maintain_temperature.script.yml
         │    (Gentle temperature maintenance)
         │    │
         │    └──> evap_set_evap_target.script.yml
         │
         └──> evap_pad_drying.script.yml
              (Pad drying with progress tracking)
              │
              └──> evap_set_evap_target.script.yml
```

## Feels-Like Temperature Calculation

```
┌─────────────────────────────────────────────────┐
│  Feels-Like Temperature Algorithm               │
├─────────────────────────────────────────────────┤
│                                                 │
│  fan_fraction = (fan_speed - 1) / 9             │
│                                                 │
│  1. Calculate Wet Bulb (Stull approximation):   │
│     Twb from outdoor temp & RH                  │
│                                                 │
│  2. Predict Supply Temperature:                 │
│     eta = 0.75 + (0.55 - 0.75) × fan_frac      │
│     supply_heat_gain = f(outdoor_temp)          │
│     Ts = outdoor_temp - eta×(outdoor - Twb)     │
│          + supply_heat_gain                     │
│                                                 │
│  3. Mixing Model:                               │
│     mix_coeff = 0.55 × fan_frac                 │
│     T_eff = T_indoor - mix_coeff×(T_in - Ts)   │
│                                                 │
│  4. Humidity Adjustment (cool mode):            │
│     RH_eff = RH_indoor + 6.0 × fan_frac        │
│                                                 │
│  5. Air Velocity:                               │
│     v = 0.10 + fan_frac × 1.40 m/s             │
│                                                 │
│  6. Steadman Apparent Temperature:              │
│     AT = T_eff + 0.33×e - 0.70×v - 4.0         │
│     where e = vapor pressure from RH_eff        │
│                                                 │
└─────────────────────────────────────────────────┘
```

## Pad Drying Progress Tracking

```
┌──────────────────────────────────────────────┐
│  Pad Drying Progress (0-100%)                │
├──────────────────────────────────────────────┤
│                                              │
│  Drying Time Interpolation:                 │
│    dry_time = speed1_time + (speed - 1) ×   │
│               (speed10_time - speed1_time)/9 │
│                                              │
│  Default values:                             │
│    speed1_time = 10 minutes                  │
│    speed10_time = 5 minutes                  │
│                                              │
│  Progress calculation:                       │
│    percent_delta = (elapsed_min / dry_time) × 100% │
│    progress = min(100%, current% + percent_delta)  │
│                                              │
│  Fan speed reduction:                        │
│    - Lower fan when progress >= 80%          │
│    - Lower fan when indoor < setpoint        │
│    - Turn off when progress = 100%           │
│                                              │
└──────────────────────────────────────────────┘
```

## Fan Speed Constraints (v4)

```
┌─────────────────────────────────────────────┐
│  Set Evap Target Safety Constraints         │
├─────────────────────────────────────────────┤
│                                             │
│  Maximum change per cycle: ±2 steps         │
│  Fan speed range: 1-10                      │
│  Min/max speed entities: optional override  │
│                                             │
│  Transition to OFF:                         │
│    If fan >= 2: reduce by 2, switch to     │
│                 fan_only, then off          │
│    If fan < 2:  go directly to off          │
│                                             │
│  Retry logic: up to 15 attempts per change  │
│  Wait time: 2 seconds between attempts      │
│                                             │
└─────────────────────────────────────────────┘
```

## Maintain Temperature Logic (v4)

```
┌────────────────────────────────────────────────┐
│  Maintain Temperature Script Decision         │
├────────────────────────────────────────────────┤
│                                                │
│  If temperature_change_per_minute available:   │
│    - diff > deadband_upper × 2: fan +1         │
│    - diff < deadband_lower × 2: fan -1         │
│    - cooling & diff < 0: fan -1                │
│    - warming & diff > 0: fan +1                │
│                                                │
│  Otherwise:                                    │
│    - Test fan speeds: current-1, current, +1   │
│    - Calculate feels-like at each speed        │
│    - Select speed closest to setpoint          │
│                                                │
│  Mode selection:                               │
│    - Switch to fan_only if:                    │
│      * not at_target AND                       │
│      * diff < -0.9°C AND                       │
│      * current_fan_speed <= 2                  │
│    - Otherwise: stay in cool mode              │
│                                                │
└────────────────────────────────────────────────┘
```

## Get to Target Logic (v4)

```
┌────────────────────────────────────────────────┐
│  Get to Target Temperature Script             │
├────────────────────────────────────────────────┤
│                                                │
│  Target Time Calculation:                      │
│    target_minutes = max(3, 3 - (diff × 0.5))   │
│                                                │
│  For each fan speed (1-10):                    │
│    1. Calculate feels-like temperature         │
│    2. Estimate cooling rate                    │
│    3. Calculate time to reach target           │
│    4. Find speed closest to target_minutes     │
│                                                │
│  Apply ±2 step constraint from current speed   │
│                                                │
│  Always operates in cool mode                  │
│                                                │
└────────────────────────────────────────────────┘
```

## Integration Points (v4)

```
┌────────────────────────────────────────────────────┐
│               External Interfaces                  │
├────────────────────────────────────────────────────┤
│                                                    │
│  Input Entities:                                   │
│    • sensor.home_temperature    (Indoor temp)      │
│    • sensor.viewbank_temp       (Outdoor temp)     │
│    • sensor.viewbank_humidity   (Outdoor/Indoor RH)│
│    • input_number.target_temp   (User setpoint)    │
│    • input_boolean.auto_control (Enable/Disable)   │
│    • input_number.pad_drying_progress (0-100%)     │
│                                                    │
│  Output Actions:                                   │
│    • climate.set_hvac_mode      (off/fan_only/cool)│
│    • climate.set_fan_mode       (1-10)            │
│    • input_number.set_value     (pad progress)     │
│                                                    │
│  Scripts Called:                                   │
│    • evap_get_to_target_temperature               │
│    • evap_maintain_temperature                    │
│    • evap_pad_drying                              │
│    • evap_set_evap_target                         │
│                                                    │
│  Features Removed from v3:                         │
│    • sensor.home_feels_like (calculated inline)    │
│    • sensor.wet_bulb_temperature (calculated inline)│
│    • Indoor humidity estimation                    │
│    • Time-based pad drying                        │
│                                                    │
└────────────────────────────────────────────────────┘
```

## Key Thresholds Summary (v4)

```
┌─────────────────────────────────────────────────┐
│              Control Thresholds (v4)            │
├─────────────────────────────────────────────────┤
│                                                 │
│ maintain_threshold:         1.0°C (default)     │
│ deadband_cooling_upper:     0.3°C (default)     │
│ deadband_cooling_lower:     0.0°C (default)     │
│                                                 │
│ Get to target mode:         diff > 1.0°C        │
│ Maintain temperature mode:  diff <= 1.0°C       │
│                                                 │
│ Pad drying start:           diff < 1.0°C        │
│ Pad drying complete:        progress = 100%     │
│ Fan speed reduction:        progress >= 80%     │
│                                                 │
│ Switch to fan_only:         diff < -0.9°C AND   │
│                             fan_speed <= 2      │
│                                                 │
│ Max fan speed change:       ±2 steps per cycle  │
│ Temperature change reliable: elapsed > 5 sec    │
│                                                 │
└─────────────────────────────────────────────────┘
```

## Error Handling (v4)

```
┌─────────────────────────────────────────────────┐
│              Crash Protection                   │
├─────────────────────────────────────────────────┤
│                                                 │
│  Script calls: continue_on_error: true          │
│    - evap_maintain_temperature                  │
│    - evap_get_to_target_temperature             │
│    - evap_pad_drying                            │
│                                                 │
│  Automation mode: restart                       │
│    - Restarts if manual trigger received        │
│    - Ensures automation always runs when enabled│
│                                                 │
│  State restoration:                             │
│    - Snapshot taken at automation start         │
│    - Restored when auto mode disabled           │
│                                                 │
│  Default values for sensors:                    │
│    - Indoor temp: 22°C                          │
│    - Outdoor temp: 30°C                         │
│    - Humidity: 50%                              │
│    - Temperature change: -999 (unavailable)     │
│                                                 │
└─────────────────────────────────────────────────┘
```

## v4 Improvements Over v3

```
┌─────────────────────────────────────────────────┐
│           Key v4 Enhancements                   │
├─────────────────────────────────────────────────┤
│                                                 │
│ Removed:                                        │
│  ✗ Indoor humidity estimation (unreliable)      │
│  ✗ Separate wet bulb sensor requirement        │
│  ✗ Time-based pad drying (5-10 min fixed)      │
│  ✗ Complex maintain_mode flag logic            │
│                                                 │
│ Added:                                          │
│  ✓ Pad drying progress tracking (0-100%)       │
│  ✓ Inline feels-like temperature calculation    │
│  ✓ Temperature change rate tracking             │
│  ✓ Crash protection (continue_on_error)         │
│  ✓ Clearer get-to-target vs maintain separation│
│  ✓ Fan speed constraint enforcement (±2 steps)  │
│                                                 │
│ Improved:                                       │
│  ↑ Control algorithm clarity                    │
│  ↑ Fan speed stability (consistent goal)        │
│  ↑ Error handling and robustness               │
│  ↑ Predictive control with temp change rate     │
│                                                 │
└─────────────────────────────────────────────────┘
```
