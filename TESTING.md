# Testing Guide for Maintain Temperature Mode

## Overview
This implementation adds a "maintain temperature mode" that provides stable fan speed control when the evaporative cooler reaches the target temperature. This document explains how to test the new functionality.

## Prerequisites

### Required Entities
Ensure the following entities are configured in your Home Assistant:
- `climate.evap` - Your evaporative cooler
- `sensor.home_temperature` - Indoor temperature sensor
- `input_number.target_temperature` - Target temperature control
- `input_boolean.automatic_control` - Automation control toggle
- `sensor.viewbank_temp` - Outdoor temperature sensor (adjust name in automation)
- `sensor.viewbank_humidity` - Outdoor humidity sensor (adjust name in automation)

### Install Configuration Files
1. Add `sensor.evap_twb_viewbank.yml` to your Home Assistant sensor configuration
2. Add `evap_maintain_temperature.script.yml` to your scripts directory
3. Update `evap_apply_mode_and_fan_with_twb.script.yml` (already supports the sensor)
4. Add/update `evap-control.automation.yml` to your automations directory
5. Restart Home Assistant or reload the relevant configurations

## Test Scenarios

### Test 1: Initial Cooling to Target
**Purpose**: Verify aggressive cooling when far from target, then switch to maintain mode

**Steps**:
1. Set `input_number.target_temperature` to 24°C
2. Ensure indoor temperature is significantly above target (e.g., 27°C)
3. Enable `input_boolean.automatic_control`
4. Observe the system behavior

**Expected Results**:
- System turns on in `cool` mode
- Fan speed increases aggressively based on temperature difference
- When temperature reaches ~24.2°C (within 0.3°C), maintain mode activates
- Fan speed changes become minimal (only ±1 step when error > 0.2°C)

**Verification**:
- Check automation traces in Home Assistant
- Look for `maintain_mode` variable transitions (false → true)
- Monitor fan speed changes (should be frequent initially, then stabilize)

### Test 2: Maintaining Temperature
**Purpose**: Verify stable fan operation when at target

**Steps**:
1. Start with temperature at target (e.g., 24.1°C, setpoint 24°C)
2. System should be in maintain mode
3. Observe fan speed over 10-15 minutes

**Expected Results**:
- Fan speed remains constant if temperature stays within ±0.2°C of setpoint
- Only small adjustments (±1 speed) if temperature varies by > 0.2°C
- No rapid cycling or frequent changes

**Verification**:
- Use Home Assistant's history graph for `climate.evap` fan_mode
- Should show stable horizontal line with occasional single-step adjustments

### Test 3: Temperature Rise in Maintain Mode
**Purpose**: Verify exit from maintain mode when temperature rises significantly

**Steps**:
1. Start in maintain mode at target temperature
2. Simulate temperature rise (e.g., open windows/doors, or wait for natural heat gain)
3. Let temperature exceed setpoint by > 0.3°C

**Expected Results**:
- When temp > setpoint + 0.3°C, system exits maintain mode
- Aggressive cooling resumes with larger fan speed increases
- Once target reached again, maintain mode re-activates

**Verification**:
- Check automation traces showing `maintain_mode` transition (true → false → true)
- Fan speed should increase more rapidly during aggressive phase

### Test 4: Below Target with Fan Speed > 1
**Purpose**: Verify intelligent transition when temperature falls below target

**Steps**:
1. Cool to below setpoint (e.g., 23.5°C, setpoint 24°C)
2. Ensure current fan speed is 3 or higher
3. Wait 5 minutes

**Expected Results**:
- After 5 minutes below target, fan speed reduces to 1
- System stays in `cool` mode (NOT fan_only)
- Maintain mode activates

**Verification**:
- Fan should change to speed 1
- HVAC mode should remain `cool`
- `maintain_mode` should be true

### Test 5: Below Target at Fan Speed 1
**Purpose**: Verify transition to fan_only for pad drying

**Steps**:
1. Cool to below setpoint with fan already at speed 1
2. Wait 5 minutes

**Expected Results**:
- After 5 minutes below target at fan 1, switches to `fan_only`
- Fan speed ramps down over 5 minutes (or 2 minutes in aggressive mode)
- Maintains fan_only for minimum 5 minutes
- After 10 minutes total, turns off

**Verification**:
- HVAC mode changes from `cool` to `fan_only`
- Fan speed gradually decreases to 1
- Eventually system turns `off`

### Test 6: Wet Bulb Temperature Sensor
**Purpose**: Verify the Twb sensor is working correctly

**Steps**:
1. Check if `sensor.evap_twb_viewbank` entity exists
2. Verify it has a valid temperature value
3. Compare with manual calculation or online wet bulb calculator

**Expected Results**:
- Sensor should show temperature between outdoor temp and dew point
- Value should be lower than outdoor dry-bulb temperature
- Should update when outdoor temp/humidity changes

**Verification**:
- Use Developer Tools → States to view `sensor.evap_twb_viewbank`
- Check sensor attributes and history

### Test 7: Setpoint Change in Maintain Mode
**Purpose**: Verify maintain mode adjusts properly to new setpoint

**Steps**:
1. Start in maintain mode at 24°C
2. Change setpoint to 23°C
3. Observe system response

**Expected Results**:
- If new setpoint is near current temp, system stays in maintain mode
- If new setpoint requires significant cooling, exits maintain mode
- Fan adjusts appropriately based on new target

## Monitoring and Debugging

### Key Indicators
1. **Maintain Mode Status**: Check automation traces for `maintain_mode` variable
2. **Fan Speed Changes**: Monitor `climate.evap` fan_mode attribute
3. **Temperature Trend**: Use history graph for `sensor.home_temperature`
4. **Script Calls**: Look for calls to `evap_maintain_temperature` vs `evap_apply_mode_and_fan_with_twb`

### Common Issues

**Issue**: Fan speed changes too frequently in maintain mode
- Check if temperature sensor is stable
- Verify fan_change_threshold (default 0.2°C)
- Look for external factors causing temperature fluctuations

**Issue**: System doesn't enter maintain mode
- Verify `at_target` condition (temp within ±0.3°C of setpoint)
- Check if system is in `cool` mode
- Review automation traces for maintain_mode transitions

**Issue**: Wet bulb sensor shows "unavailable"
- Verify `sensor.viewbank_temp` and `sensor.viewbank_humidity` are valid
- Check sensor configuration syntax
- Ensure humidity value is > 0

## Success Criteria

The implementation is successful if:
1. ✓ System aggressively cools when far from target
2. ✓ Maintains stable fan speed when at target (±0.2°C)
3. ✓ Only switches to fan_only when at fan speed 1 and below target
4. ✓ Fan speed changes are minimal in maintain mode (±1 step max)
5. ✓ Wet bulb temperature sensor provides valid values
6. ✓ System responds appropriately to temperature and setpoint changes

## Notes

- Temperature sensor noise can affect performance. Consider using a filtered sensor if readings fluctuate rapidly.
- Outdoor conditions (humidity, temperature) significantly impact evaporative cooler effectiveness.
- The 0.2°C threshold for maintain mode can be adjusted via the `fan_change_threshold` parameter in the maintain script if needed.
- Aggressive mode for fan ramp-down activates when outdoor temp > setpoint + 2°C.
