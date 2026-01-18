# System Architecture and Flow Diagrams

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Home Assistant                           │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │         Evaporative Cooler Automation                │ │
│  │                                                       │ │
│  │  ┌────────────┐         ┌──────────────┐           │ │
│  │  │  Sensors   │────────>│ Control Loop │           │ │
│  │  │            │         │              │           │ │
│  │  │ • Home Temp│         │ • Monitor    │           │ │
│  │  │ • Outdoor  │         │ • Decide     │           │ │
│  │  │ • Humidity │         │ • Execute    │           │ │
│  │  │ • Twb      │         │              │           │ │
│  │  └────────────┘         └──────┬───────┘           │ │
│  │                                 │                    │ │
│  │                    ┌────────────┴───────────┐       │ │
│  │                    │                        │       │ │
│  │              ┌─────▼──────┐          ┌─────▼──────┐│ │
│  │              │ Aggressive │          │  Maintain  ││ │
│  │              │  Cooling   │          │Temperature ││ │
│  │              │   Script   │          │   Script   ││ │
│  │              └─────┬──────┘          └─────┬──────┘│ │
│  │                    │                        │       │ │
│  │                    └────────────┬───────────┘       │ │
│  └────────────────────────────────┼───────────────────┘ │
│                                    │                     │
│                         ┌──────────▼──────────┐         │
│                         │  climate.evap       │         │
│                         │  (HVAC Device)      │         │
│                         └─────────────────────┘         │
└─────────────────────────────────────────────────────────┘
```

## State Machine Diagram

```
                    ┌─────────────────────┐
                    │                     │
                    │       OFF           │
                    │                     │
                    └──────────┬──────────┘
                               │
                  need_cooling │
                               │
                    ┌──────────▼──────────┐
                    │                     │
                    │   COOL (Aggressive) │
                    │   maintain_mode=F   │
                    │                     │
                    └──────────┬──────────┘
                               │
                   at_target   │
                               │
                    ┌──────────▼──────────┐
       ┌────────────┤                     │
       │            │   COOL (Maintain)   │
       │            │   maintain_mode=T   │
       │            │                     │
       │            └──────────┬──────────┘
       │                       │
       │                       │ temp exceeds
       │     need_cooling      │ threshold
       │        again          │ by 0.2°C
       │                       │
       │            ┌──────────▼──────────┐
       │            │                     │
       │            │   COOL (Adjust)     │
       │            │   maintain_mode=T   │
       │            │   Fan +/- 1         │
       │            │                     │
       │            └──────────┬──────────┘
       │                       │
       └───────────────────────┘
                               │
         below_5m + fan_speed=1│
                               │
                    ┌──────────▼──────────┐
                    │                     │
                    │     FAN_ONLY        │
                    │   (Pad Drying)      │
                    │                     │
                    └──────────┬──────────┘
                               │
                      below_10m│
                               │
                    ┌──────────▼──────────┐
                    │                     │
                    │       OFF           │
                    │                     │
                    └─────────────────────┘
```

## Temperature Error vs. Fan Speed Response

### Aggressive Cooling Mode (maintain_mode = false)
```
Temperature Error (°C)    Fan Speed Change
─────────────────────────────────────────
     > 3.0                   +10 (max)
     2.0 - 3.0               +4 boost
     1.0 - 2.0               +2 boost
     0.6 - 1.0               +1 boost
     0.3 - 0.6               Twb calculation
     < 0.3                   Switch to maintain
```

### Maintain Temperature Mode (maintain_mode = true)
```
Temperature Error (°C)    Fan Speed Change
─────────────────────────────────────────
     > 0.8                   Exit to aggressive
     0.2 - 0.8               +1 step
    -0.2 - 0.2               No change
    -0.8 - -0.2              -1 step
     < -0.8                  Exit to below_target
```

## Decision Flow Chart

```
Temperature reading arrives
          │
          ▼
    ┌─────────┐
    │ diff =  │
    │ T - S   │
    └────┬────┘
         │
    ┌────▼─────────────────────────┐
    │ diff > 0.3°C?               │
    │ (need_cooling)               │
    └────┬─────────────────┬───────┘
         │ YES             │ NO
         ▼                 ▼
  ┌──────────────┐   ┌───────────────┐
  │ maintain=F   │   │ -0.3 < diff   │
  │ Aggressive   │   │ < 0.3?        │
  │ Cooling      │   │ (at_target)   │
  └──────────────┘   └───┬───────────┘
                         │ YES  │ NO
                         ▼      ▼
                  ┌──────────┐ ┌─────────┐
                  │maintain=T│ │ Below   │
                  │Stable    │ │ Target  │
                  │Control   │ │ Logic   │
                  └──────────┘ └─────────┘
```

## Fan Speed Adjustment Logic (Maintain Mode)

```
┌─────────────────────────────────────────┐
│ Temperature Error = Tin - Setpoint      │
└───────────────┬─────────────────────────┘
                │
        ┌───────┴───────┐
        │               │
   err > 0.2°C?    err < -0.2°C?
        │               │
    ┌───▼───┐       ┌───▼───┐       ┌────────┐
    │ YES   │       │ YES   │       │   NO   │
    └───┬───┘       └───┬───┘       └───┬────┘
        │               │               │
        ▼               ▼               ▼
 ┌────────────┐  ┌────────────┐  ┌────────────┐
 │ Increase   │  │ Decrease   │  │ Maintain   │
 │ fan by 1   │  │ fan by 1   │  │ current    │
 │ (max 10)   │  │ (min 1)    │  │ speed      │
 └────────────┘  └────────────┘  └────────────┘
```

## Below Target Decision Tree

```
Temperature below setpoint for 5 minutes
                    │
          ┌─────────┴─────────┐
          │                   │
    Current fan > 1?    Current fan = 1?
          │                   │
          ▼                   ▼
  ┌───────────────┐   ┌──────────────┐
  │ Set fan to 1  │   │ Switch to    │
  │ Stay in COOL  │   │ FAN_ONLY     │
  │ maintain=true │   │ maintain=F   │
  └───────────────┘   └──────────────┘
          │                   │
          ▼                   ▼
  ┌───────────────┐   ┌──────────────┐
  │ Continue      │   │ Ramp down    │
  │ with minimal  │   │ over 5 min   │
  │ adjustments   │   │ (pad drying) │
  └───────────────┘   └──────────────┘
```

## Timing Diagram Example

```
Time →
─────────────────────────────────────────────────────────────►

Temp:  27°C ─────────▼─────────▼─────────▼────────
                     24.5°C   24.2°C   24.0°C
       
Mode:  AGGRESSIVE ───────────┬────────────────────
                        MAINTAIN (stable)
       
Fan:   8 ──┬─── 6 ──┬─── 4 ──┬─── 3 ──────────────
           │        │        │        
           3m       2m       2m       (no changes)
```

## Integration Points

```
┌────────────────────────────────────────────────────┐
│               External Interfaces                  │
├────────────────────────────────────────────────────┤
│                                                    │
│  Input Entities:                                   │
│    • sensor.home_temperature    (Indoor temp)      │
│    • sensor.viewbank_temp       (Outdoor temp)     │
│    • sensor.viewbank_humidity   (Outdoor RH)       │
│    • sensor.evap_twb_viewbank   (Calculated Twb)   │
│    • input_number.target_temp   (User setpoint)    │
│    • input_boolean.auto_control (Enable/Disable)   │
│                                                    │
│  Output Actions:                                   │
│    • climate.set_hvac_mode      (off/fan_only/cool)│
│    • climate.set_fan_mode       (1-10)            │
│                                                    │
│  Scripts Called:                                   │
│    • evap_apply_mode_and_fan_with_twb             │
│    • evap_maintain_temperature                    │
│    • evap_ramp_fan_in_fan_only                    │
│                                                    │
└────────────────────────────────────────────────────┘
```

## Key Thresholds Summary

```
┌─────────────────────────────────────────────────┐
│              Control Thresholds                 │
├─────────────────────────────────────────────────┤
│                                                 │
│ Enter aggressive cooling:  diff > 0.3°C        │
│ Enter maintain mode:        -0.3 ≤ diff ≤ 0.3  │
│ Maintain adjustment:        |err| > 0.2°C      │
│ Below target detection:     diff < 0.0°C       │
│ Below for 5m:               Reduce fan / fan_only│
│ Below for 10m:              Turn off            │
│ Pad drying minimum:         5 minutes           │
│ Temperature fall exception: diff < -1.0°C       │
│                                                 │
└─────────────────────────────────────────────────┘
```
