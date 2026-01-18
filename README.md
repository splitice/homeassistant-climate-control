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
*Set up variables for temperature, setpoint, difference, need for cooling, HVAC state, and fan speed.*

**R5. If cooling is needed, ensure HVAC is in cool mode with wet bulb optimized fan speed**  
*If the temperature is more than 0.3°C above setpoint, switch to cool mode and set fan speed optimized based on outdoor wet bulb temperature calculation.*

**R6. If cooling is not needed but HVAC is still in cool, switch to fan_only to dry pads**  
*If the system is cooling but temperature crosses below setpoint, switch to fan-only mode to allow evaporative pads to dry for a minimum of 5 minutes (unless temperature falls more than 1°C below target).*

**R7. Main persistent control loop**  
*Continuously monitor temperature, setpoint, and control state, responding to changes as needed.*

**R8. If auto turned off, restore snapshot and exit loop**  
*If automatic control is turned off during the loop, restore the previous state and exit.*

**R9. Track how long temperature is below setpoint**  
*Update the timer for how long the temperature is below the setpoint.*

**R10. Calculate how long we've been below setpoint (5m/10m thresholds)**  
*Determine if the temperature has been below setpoint for 5 or 10 minutes to trigger further actions.*

**R11. If off and need cooling, turn on with wet bulb optimized fan speed**  
*If the system is off but cooling is needed, turn it on and set the fan speed optimized based on outdoor wet bulb temperature calculation.*

**R12. If in fan_only and need cooling, switch to cool with wet bulb optimized fan (respecting drying cycle)**  
*If in fan-only mode but cooling is needed, switch to cool mode with fan speed calculated based on outdoor wet bulb temperature. However, if in a drying cycle (less than 5 minutes since entering fan_only and temperature has not fallen 1°C below target), remain in fan_only mode until drying conditions are met.*

**R13. If below setpoint for 10m, turn off**  
*If the temperature has been below setpoint for 10 minutes, turn the system off.*

**R14. If below setpoint for 5m, switch from cool to fan_only with current fan speed**  
*If the temperature has been below setpoint for 5 minutes, switch from cool to fan-only mode maintaining the current fan speed, which will then be gradually ramped down over 5 minutes (see R18).*

**R15-R16. Adjust fan speed based on wet bulb temperature (not during drying cycle)**  
*Continuously optimize fan speed based on outdoor wet bulb temperature, target indoor temperature, and thermodynamic efficiency curves. Adjustments are prevented during the evaporative pad drying cycle. The wet bulb calculation considers outdoor temperature and humidity to determine the optimal saturation efficiency and corresponding fan speed.*

**R17. If setpoint changed and now near target, adjust fan with wet bulb calculation**  
*If the setpoint is changed and the temperature is now near the target, adjust the fan speed based on wet bulb temperature calculation.*

**R18. Ramp down fan speed in fan_only mode**  
*When in fan_only mode, gradually reduce the fan speed from the initial speed (when entering fan_only, capped at maximum of 6) to speed 1 over a configurable duration (default 5 minutes). Supports aggressive mode that completes ramp-down in 2 minutes when outdoor temperature exceeds setpoint by more than 2°C. This provides a smooth transition that continues the pad drying process while reducing energy consumption and noise.*

---

## Design Rationale

- **Snapshot/restore (R1, R2, R8):** Ensures user settings are preserved and system can safely exit automation.
- **Persistent loop (R7):** Allows for robust, real-time response to environment and user changes.
- **Wet bulb temperature-based fan control (R5, R11, R12, R15-R17):** Uses outdoor wet bulb temperature (Twb) calculation to optimize fan speed based on thermodynamic efficiency. The system calculates Twb using the Stull approximation from outdoor dry-bulb temperature and relative humidity, then determines the required saturation efficiency to reach the target temperature. Fan speed is selected to match this efficiency, maximizing cooling effectiveness while minimizing energy use.
- **Pad drying cycle (R6, R12, R15-R16, R18):** When temperature crosses below setpoint, maintains fan_only mode for 5 minutes minimum to dry evaporative pads, preventing mold growth and extending pad lifespan. Early exit allowed if temperature falls >1°C below target. During the fan_only period, fan speed is gradually ramped down from the initial speed (capped at 6) to speed 1 over a configurable duration (default 5 minutes, or 2 minutes in aggressive mode when outdoor temperature is >2°C above setpoint), providing energy savings and reduced noise while maintaining effective pad drying.
- **Below setpoint timers (R3, R9, R10, R13, R14):** Prevents rapid cycling and allows for graceful ramp-down and shutoff.
- **Variable initialization (R4):** Ensures all logic operates on up-to-date, consistent state.

## Required Entities

The automation requires the following Home Assistant entities to be configured:

- `climate.evap` - The evaporative cooler climate entity
- `sensor.home_temperature` - Indoor temperature sensor
- `input_number.target_temperature` - Target temperature setpoint
- `input_boolean.automatic_control` - Toggle for automatic control
- `sensor.viewbank_temp` - Outdoor dry-bulb temperature sensor (example name - replace with your sensor)
- `sensor.viewbank_humidity` - Outdoor relative humidity sensor (example name - replace with your sensor)

**Note**: The outdoor sensor entity names (`sensor.viewbank_temp` and `sensor.viewbank_humidity`) are location-specific examples. Update the `outdoor_temp_entity` and `outdoor_rh_entity` variables in the automation to match your actual outdoor sensor entity IDs.

## Wet Bulb Temperature Fan Control

The automation uses the `evap_apply_mode_and_fan_with_twb` script to calculate optimal fan speeds based on outdoor conditions. This script:

1. Calculates wet bulb temperature (Twb) using the Stull approximation from outdoor temperature and humidity
2. Determines the required saturation efficiency: η_req = (B - A_aim) / (B - Twb)
   - B = outdoor dry-bulb temperature
   - A_aim = target temperature + aim_offset (default 0.3°C)
   - Twb = wet bulb temperature
3. Maps efficiency to fan speed using a linear interpolation between η₁ (0.85 at fan speed 1) and η₁₀ (0.65 at fan speed 10)
4. Returns a fan speed between 1-10, or 10 if outdoor conditions are too humid (B - Twb < 1.0°C)

This approach ensures the evaporative cooler operates at the most efficient fan speed for current outdoor conditions, reducing energy consumption while maintaining comfort.

See inline comments in `evap-control.automation` for mapping of requirements to code.
