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

**R5. If cooling is needed, ensure HVAC is in cool mode and fan is high**  
*If the temperature is more than 0.5°C above setpoint, switch to cool mode and set fan to maximum.*

**R6. If cooling is not needed but HVAC is still in cool, reduce fan or turn off**  
*If the system is cooling but no longer needs to, reduce fan speed or switch to fan-only mode.*

**R7. Main persistent control loop**  
*Continuously monitor temperature, setpoint, and control state, responding to changes as needed.*

**R8. If auto turned off, restore snapshot and exit loop**  
*If automatic control is turned off during the loop, restore the previous state and exit.*

**R9. Track how long temperature is below setpoint**  
*Update the timer for how long the temperature is below the setpoint.*

**R10. Calculate how long we've been below setpoint (5m/10m thresholds)**  
*Determine if the temperature has been below setpoint for 5 or 10 minutes to trigger further actions.*

**R11. If off and need cooling, turn on and set fan high**  
*If the system is off but cooling is needed, turn it on and set the fan to maximum.*

**R12. If in fan_only and need cooling, switch to cool and restore fan**  
*If in fan-only mode but cooling is needed, switch to cool mode and restore previous fan speed.*

**R13. If below setpoint for 10m, turn off**  
*If the temperature has been below setpoint for 10 minutes, turn the system off.*

**R14. If below setpoint for 5m, switch from cool to fan_only**  
*If the temperature has been below setpoint for 5 minutes, switch from cool to fan-only mode.*

**R15. Calculate fan step based on temperature difference**  
*Adjust the fan speed up or down based on how far the temperature is from the setpoint.*

**R16. Adjust fan speed up/down as needed, with safe bounds**  
*Ensure fan speed changes are within safe limits and avoid setting below minimum.*

**R17. If setpoint changed and now near target, set fan to 1**  
*If the setpoint is changed and the temperature is now near the target, set the fan to minimum speed.*

---

## Design Rationale

- **Snapshot/restore (R1, R2, R8):** Ensures user settings are preserved and system can safely exit automation.
- **Persistent loop (R7):** Allows for robust, real-time response to environment and user changes.
- **Fan speed logic (R5, R6, R11, R12, R15, R16, R17):** Maximizes cooling efficiency and comfort, while minimizing unnecessary fan use.
- **Below setpoint timers (R3, R9, R10, R13, R14):** Prevents rapid cycling and allows for graceful ramp-down and shutoff.
- **Variable initialization (R4):** Ensures all logic operates on up-to-date, consistent state.

See inline comments in `evap-control.automation` for mapping of requirements to code.
