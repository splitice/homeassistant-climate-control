# Home Assistant Evaporative Cooler Automation

This automation controls an evaporative cooler to maintain a target temperature, with robust handling for state transitions, fan speed, and snapshot restore.

## Numbered Requirements

**R1. Restore state and exit if automatic control is off**  
*If the automation is triggered but `input_boolean.automatic_control` is off, restore the previous state and stop further actions.*

**R2. Take a snapshot of the evap state**  
*On startup, save the current state of the cooler for later restoration.*

**R3. Track how long temperature is below setpoint**  
*Initialize and update a timer to measure how long the temperature has been below the setpoint.*

**R4. Initialize main loop variables**  
*Set up variables for temperature, setpoint, difference, need for cooling, HVAC state, fan speed, and maintain mode flag.*

**R5. If cooling is needed, ensure HVAC is in cool mode with wet bulb optimized fan speed**  
*If the temperature is more than 0.3°C above setpoint, switch to cool mode and set fan speed optimized based on outdoor wet bulb temperature from sensor.*

**R6. If cooling is not needed but HVAC is still in cool, switch to fan_only to dry pads (only if at fan speed 1)**  
*If the system is cooling at fan speed 1 and temperature crosses below setpoint, switch to fan-only mode to allow evaporative pads to dry for a minimum of 5 minutes (unless temperature falls more than 1°C below target). Fan speed ramping is applied immediately upon entering fan_only mode. If fan speed is above 1, reduce to fan speed 1 and enter maintain mode instead.*

**R7. Main persistent control loop**  
*Continuously monitor temperature, setpoint, and control state, responding to changes as needed.*

**R8. If auto turned off, restore snapshot and exit loop**  
*If automatic control is turned off during the loop, restore the previous state and exit.*

**R9. Track how long temperature is below setpoint**  
*Update the timer for how long the temperature is below the setpoint.*

**R10. Calculate how long we've been below setpoint (5m/10m thresholds)**  
*Determine if the temperature has been below setpoint for 5 or 10 minutes to trigger further actions.*

**R11. If off and need cooling, turn on with wet bulb optimized fan speed**  
*If the system is off but cooling is needed, turn it on and set the fan speed optimized based on outdoor wet bulb temperature from sensor.*

**R12. If in fan_only and need cooling, switch to cool with wet bulb optimized fan (respecting drying cycle)**  
*If in fan-only mode but cooling is needed, switch to cool mode with fan speed calculated based on outdoor wet bulb temperature from sensor. However, if in a drying cycle (less than 5 minutes since entering fan_only and temperature has not fallen 1°C below target), remain in fan_only mode until drying conditions are met.*

**R13. If below setpoint for 10m, turn off**  
*If the temperature has been below setpoint for 10 minutes, turn the system off.*

**R14. If below setpoint for 5m, intelligently transition based on fan speed**  
*If the temperature has been below setpoint for 5 minutes: If fan speed is above 1, reduce to fan speed 1 and enter maintain mode. Only switch to fan-only mode for pad drying if already at fan speed 1 and temperature is still below setpoint.*

**R15-R16. Adjust fan speed based on maintain mode**  
*When in maintain mode (temperature at target), use the maintain temperature script to make minimal fan speed adjustments (±1 step max) based on temperature error threshold (0.2°C). When not in maintain mode (actively cooling), use aggressive fan speed adjustments with wet bulb calculation. Adjustments are prevented during the evaporative pad drying cycle.*

**R17. If setpoint changed and now near target, adjust fan with wet bulb calculation**  
*If the setpoint is changed and the temperature is now near the target, adjust the fan speed based on wet bulb temperature calculation.*

**R18. Ramp down fan speed in fan_only mode**  
*When in fan_only mode, gradually reduce the fan speed from the initial speed (when entering fan_only, capped at maximum of 6) to speed 1 over a configurable duration (default 5 minutes). Supports aggressive mode that completes ramp-down in 2 minutes when outdoor temperature exceeds setpoint by more than 2°C. This provides a smooth transition that continues the pad drying process while reducing energy consumption and noise.*

---

## Design Rationale

- **Snapshot/restore (R1, R2, R8):** Ensures user settings are preserved and system can safely exit automation.
- **Persistent loop (R7):** Allows for robust, real-time response to environment and user changes.
- **Maintain temperature mode (R15-R16):** When temperature reaches target (within 0.3°C), switches to maintain mode which uses minimal fan speed adjustments (±1 step maximum) with a temperature error threshold (0.2°C) to prevent frequent cycling. This provides stable, quiet operation while maintaining comfort.
- **Wet bulb temperature sensor (sensor.viewbank_wet_bulb_temperature):** Pre-calculated wet bulb temperature using the Stull approximation from outdoor dry-bulb temperature and relative humidity. Used by both aggressive cooling and maintain temperature scripts for efficiency calculations.
- **Intelligent fan_only transition (R6, R14):** Fan_only mode for pad drying is only activated when the system is at fan speed 1 and temperature is below setpoint. If fan speed is above 1, the system reduces to fan speed 1 and enters maintain mode instead, allowing the evaporative cooler to continue providing efficient cooling.
- **Pad drying cycle (R6, R12, R18):** When at fan speed 1 and temperature crosses below setpoint, maintains fan_only mode for 5 minutes minimum to dry evaporative pads, preventing mold growth and extending pad lifespan. Early exit allowed if temperature falls >1°C below target. Fan speed ramping is applied immediately upon entering fan_only mode (including during startup), gradually reducing the fan speed from the initial speed (capped at 6) to speed 1 over a configurable duration (default 5 minutes, or 2 minutes in aggressive mode when outdoor temperature is >2°C above setpoint), providing energy savings and reduced noise while maintaining effective pad drying.
- **Below setpoint timers (R3, R9, R10, R13, R14):** Prevents rapid cycling and allows for graceful transition to maintain mode or shutoff.
- **Variable initialization (R4):** Ensures all logic operates on up-to-date, consistent state.

## Required Entities

The automation requires the following Home Assistant entities to be configured:

- `climate.evap` - The evaporative cooler climate entity
- `sensor.home_feels_like` - Indoor "feels like" temperature sensor (accounts for humidity and evaporative cooling effect)
- `input_number.target_temperature` - Target temperature setpoint
- `input_boolean.automatic_control` - Toggle for automatic control
- `sensor.viewbank_temp` - Outdoor dry-bulb temperature sensor (example name - replace with your sensor)
- `sensor.viewbank_humidity` - Outdoor relative humidity sensor (example name - replace with your sensor)
- `sensor.viewbank_wet_bulb_temperature` - Wet bulb temperature sensor (created by sensor configuration file)

**Note**: The outdoor sensor entity names (`sensor.viewbank_temp` and `sensor.viewbank_humidity`) are location-specific examples. Update the `outdoor_temp_entity` and `outdoor_rh_entity` variables in the automation to match your actual outdoor sensor entity IDs. The `sensor.viewbank_wet_bulb_temperature` sensor should be configured using the provided `sensor.viewbank_wet_bulb_temperature.yml` file.

## Wet Bulb Temperature Sensor

The `sensor.viewbank_wet_bulb_temperature` is a template sensor that calculates the wet bulb temperature using the Stull approximation from outdoor dry-bulb temperature and relative humidity. This pre-calculated sensor is used by both the aggressive cooling script and the maintain temperature script for efficiency calculations. To configure this sensor, add the contents of `sensor.viewbank_wet_bulb_temperature.yml` to your Home Assistant configuration.

## Temperature Control Modes

The automation operates in two distinct modes:

### Aggressive Cooling Mode
Used when temperature is significantly above setpoint (>0.3°C). The `evap_apply_mode_and_fan_with_twb` script actively adjusts fan speed based on:
- Temperature error (how far above setpoint)
- Wet bulb temperature efficiency calculations
- Can make large fan speed increases (up to 4 steps) when needed

### Maintain Temperature Mode
Activated when temperature reaches target (within 0.3°C of setpoint). The `evap_maintain_temperature` script provides stable operation with:
- Minimal fan speed changes (±1 step maximum)
- Temperature error threshold (0.2°C) prevents frequent adjustments
- Maintains consistent, quiet operation
- Only changes fan speed when temperature error exceeds threshold

The system automatically switches between modes based on temperature error, providing aggressive cooling when needed and stable maintenance when at target.

## Wet Bulb Temperature Fan Control

The automation uses two scripts for fan speed control:

### evap_apply_mode_and_fan_with_twb (Aggressive Cooling)
Used during initial cooling phase when temperature is significantly above setpoint. This script:

1. Uses wet bulb temperature from `sensor.viewbank_wet_bulb_temperature` (or calculates inline if sensor not provided)
2. Determines the required saturation efficiency: η_req = (B - A_aim) / (B - Twb)
   - B = outdoor dry-bulb temperature
   - A_aim = target temperature + aim_offset (default 0.3°C)
   - Twb = wet bulb temperature
3. Maps efficiency to fan speed using a linear interpolation between η₁ (0.85 at fan speed 1) and η₁₀ (0.65 at fan speed 10)
4. Applies necessity-based boost for large temperature errors
5. Returns a fan speed between 1-10, or 10 if outdoor conditions are too humid (B - Twb < 1.0°C)

### evap_maintain_temperature (Maintain Mode)
Used when temperature is at or near setpoint for stable operation. This script:

1. Uses wet bulb temperature from `sensor.viewbank_wet_bulb_temperature` for base calculations
2. Only changes fan speed if temperature error exceeds threshold (default 0.2°C)
3. Makes minimal adjustments (±1 fan speed step maximum)
4. Maintains current fan speed when within acceptable temperature range
5. Provides quiet, stable operation with minimal cycling

This dual-mode approach ensures the evaporative cooler operates efficiently: aggressive cooling when needed, stable maintenance at target temperature.

See inline comments in `evap-control.automation` for mapping of requirements to code.
