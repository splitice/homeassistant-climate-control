# Evaporative Cooler Control Implementation (v4)

Core constaints:
 - Fan speed 1-10
 - Fan speed should never change by more than 2 steps at a time
 - A consistent fan speed should be the algorithm goal

## Main Control Loop

Description: An always running automation that monitors the evaporative cooler and temperature setpoint, adjusting the cooler's mode and fan speed as needed to maintain the desired temperature while also managing pad drying cycles.

Inputs:
- auto_entity: Boolean input entity to enable/disable automation.
- setpoint_entity: The desired temperature setpoint entity.
- evap_entity: The evaporative cooler mode select entity to control (e.g., select.roofevap1_e90248_evap_mode).
- fan_speed_entity: The evaporative cooler fan speed select entity to control (e.g., select.roofevap1_e90248_evap_fan_speed).
- pad_drying_entity: number input entity to track percent pad drying progress. (0-100%)
- outdoor_temp_entity: Optional. Outdoor temperature sensor entity for wet-bulb calculations.
- outdoor_humidity_entity: Optional. Outdoor humidity sensor entity for wet-bulb calculations.
- indoor_humidity_entity: Optional. Indoor humidity sensor entity for wet-bulb calculations. If not provided assume same as outdoor humidity.
- indoor_temperature_entity: Optional. Indoor temperature sensor entity for wet-bulb calculations.
- max_fan_speed_entity: Optional. Entity defining maximum fan speed (1-5). If not provided max fan speed is 10
- min_fan_speed_entity: Optional. Entity defining minimum fan speed (1-5). If not provided min fan speed is 1
- maintain_threshold: Optional. Temperature error (°C) to run in maintain temperature mode. Defaults to 1°C.
- pad_dry_time_speed1: Optional. Minimum time in minutes to run fan_only for pad drying. Defaults to 10 minutes.
- pad_dry_time_speed10: Optional. Minimum time in minutes to run fan_only for pad drying at speed 10. Defaults to 5 minutes.
- deadband_cooling_upper: Upper deadband (°C) for cooling decisions. Defaults to 0.3°C. (i.e., cooling starts when temp is this much above setpoint) 
- deadband_cooling_lower: Lower deadband (°C) for cooling decisions. Defaults to 0°C (cooling runs until temp is this much below setpoint)

Wet bulb Temperature Calculation:
 - If outdoor sensors are provided, calculate wet bulb temperature using outdoor temp and humidity.
 - If not provided, use indoor temperature as a proxy for wet bulb temperature.

State Variables:
 - diff: Temperature difference (current_temp - setpoint_temp).
 - needs_cooling: Boolean indicating if cooling is needed based on temperature difference and deadband (default 0 to 0.3°C).
 - at_target: Boolean indicating if current temperature is within the deadband range of the setpoint.

```
if (auto_entity is off) {
  exit automation
}
take snapshot of current evap_entity state as scene.evap_restore
set global variables {
    initialise state variables as defaults / nil
    pad_drying_previous = 0
    temperature_change_per_minute = -999
}

repeat while(auto_entity is on) {    
    set {
        setpoint_temp = state of setpoint_entity
        wet_bulb_temp = wet bulb temperature using outdoor_temp_entity, outdoor_humidity_entity
        indoor_temp_raw = state of indoor_temperature_entity
        indoor_feels_like_temp = feels like algorithm using indoor_temp_raw, indoor_humidity_entity, fan_speed
        diff = indoor_feels_like_temp - setpoint_temp
        at_target = (diff <= deadband_cooling_upper and diff >= deadband_cooling_lower)
        should_maintain = (abs(diff) <= maintain_threshold)
        needs_cooling = not (diff < deadband_cooling_lower)
        cur_mode = current state of evap_entity
        cur_fan = current fan speed of evap_entity
        alg_state = "initial"
    }
    if (should_maintain or needs_cooling) {
        set pad_drying_enabled = false
        if (should_maintain) {
            set alg_state = "maintain"
            call script evap_maintain_temperature
        } else {
            set alg_state = "get_to_target"
            call script evap_get_to_target_temperature
        }
    } else if (cur_mode == 'cool') {
        set alg_state = "pad_drying_start"
        call script evap_pad_drying with: pad_drying_previous=0, evap_entity, pad_drying_entity, pad_dry_time_speed1, pad_dry_time_speed10
        set pad_drying_previous = now
    } else if (pad_drying_enabled is true) {
        set alg_state = "pad_drying"
        call script evap_pad_drying with: pad_drying_previous, evap_entity, pad_drying_entity, pad_dry_time_speed1, pad_dry_time_speed10
        set pad_drying_previous = now
    }

    set {
        new_fan_speed = current fan speed of evap_entity
        new_fan_mode = current state of evap_entity
    }

    if (new_fan_speed != cur_fan or new_fan_mode != cur_mode) {
        log "Evap control alg {{ alg_state }} adjusted mode from {{ cur_mode }} to {{ new_fan_mode }}, fan speed from {{ cur_fan }} to {{ new_fan_speed }} due to temp diff {{ diff }}°C"

        set start_wait = now()
        
        wait for setpoint_entity, or auto_entity change for 60 seconds
    } else {
        set start_wait = now()
        
        wait for setpoint_entity, or auto_entity change for 30 seconds
    } 

    set {
        new_temperature_raw = state of indoor_temperature_entity
        elapsed_time = now() - start_wait in seconds
        temperature_change_per_minute = if elapsed_time > 5 then (new_temperature_raw - indoor_temp_raw) / (elapsed_time / 60) else -999
    } 
}

```

## Maintain Temperature Script

Description: Script to maintain the target temperature by adjusting fan speed based on temperature difference within a defined deadband with the goal of minimizing large fan speed changes while keeping temperature stable within the target range.

Inputs:
 - evap_entity: The evaporative cooler mode select entity to control (e.g., select.roofevap1_e90248_evap_mode).
 - fan_speed_entity: The evaporative cooler fan speed select entity to control (e.g., select.roofevap1_e90248_evap_fan_speed).
 - setpoint_temp
 - wet_bulb_temp
 - indoor_temperature_entity
 - indoor_humidity_entity
 - at_target
 - deadband_cooling_upper
 - deadband_cooling_lower
 - max_fan_speed_entity
 - min_fan_speed_entity
 - temperature_change_per_minute
 
Target Fan Speed Algorithm:
   Dehumidify Mode (highest priority):
    If in cool mode and indoor_temp is 3°C or more below setpoint and feels_like is less than deadband_upper from setpoint and outdoor_temp delta from setpoint is within half of indoor_temp delta, then:
     - Switch mode to fan_only
     - Decrease fan speed by 1 (capped to min fan speed of 1)
   
   If temperature_change_per_minute != -999
    If temperature difference is above deadband_upper * 2, increase fan speed by 1 (capped to max fan speed).
    else if temperature difference is below deadband_lower * 2, decrease fan speed by 1 (capped to min fan speed).
    else if temperature_change_per_minute < -0.25 (getting cooler) and diff < 0 (too cold), decrease fan speed by 1 (capped to min fan speed).
    else if temperature_change_per_minute > 0.25 (getting warmer) and diff > 0 (too warm), increase fan speed by 1 (capped to max fan speed).
   Else
    - For the fan speed -1, +1 calculate the feels like temperature factoring in wet bulb temperature and fan speed airflow (speeds 1-10 linear effect when cool mode of 0-2 degrees airflow feels like when in fan_only mode, 0-4 degrees when in cool mode).
    - Select the fan speed that results in the feels like temperature closest to the setpoint.

```
set:
  air_temp = wet bulb temperature using outdoor_temp_entity, outdoor_humidity_entity
  feels_like_temp = feels like algorithm using wet_bulb_temp, indoor_humidity_entity
  diff = feels_like_temp - setpoint_temp
  indoor_temp_delta = setpoint_temp - indoor_temp
  outdoor_temp_delta = outdoor_temp - setpoint_temp
  
  target_mode = if (in cool mode and indoor_temp_delta >= 3 and diff < deadband_upper and outdoor_temp_delta <= indoor_temp_delta/2) then fan_only
                else if (not at_target and diff < -0.9 and current_fan_speed <= 2) then fan_only 
                else cool
  target_speed = target_speed_algorithm

call set_evap_target: target_mode, target_speed, evap_entity, min_fan_speed_entity, max_fan_speed_entity
```

## Get to Target Temperature Script

Description: Script to aggressively cool to the target temperature by adjusting fan speed based on the temperature difference until the target is reached.

Notes:
 - Uses a more aggressive cooling strategy compared to the maintain temperature script.
 - The larger the temperature difference, the faster the fan speed increase.

Inputs:
- evap_entity: The evaporative cooler mode select entity to control (e.g., select.roofevap1_e90248_evap_mode).
- fan_speed_entity: The evaporative cooler fan speed select entity to control (e.g., select.roofevap1_e90248_evap_fan_speed).
- setpoint_temp: The desired temperature setpoint.
- outdoor_temp_entity
- outdoor_humidity_entity
- indoor_temperature_entity
- indoor_humidity_entity
- max_fan_speed_entity
- min_fan_speed_entity


Target Fan Speed Algorithm:
 - Calculate for each fan speed (1-10) how long it will take to reach the feels like temperature based on current temperature difference.
    - Feels like temperature can be calculated using the wet bulb temperature of the incomming air and the fan speed estimated flow rate. If outdoor sensors are not provided use actual temperature as a proxy.
    - Select the fan speed that results in reaching the target temperature closest to the target_minutes
    - This is the target_fan_speed
 - Cap target_fan_speed as no more than an increase or decrease of 2 steps from current fan speed


```
set:
  diff = indoor_temp - setpoint_temp
  target_minutes = max(3, 3-(diff * 0.5))
  target_speed = target_speed_algorithm

call set_evap_target: target_mode = cool, target_speed = target_speed, evap_entity = evap_entity, min_fan_speed_entity, max_fan_speed_entity
```

## Evap Pad Drying Script

Inputs:
- evap_entity: The evaporative cooler mode select entity to control (e.g., select.roofevap1_e90248_evap_mode).
- fan_speed_entity: The evaporative cooler fan speed select entity to control (e.g., select.roofevap1_e90248_evap_fan_speed).
- pad_drying_entity: time sensor entity to track pad drying duration.
- pad_dry_time_speed1: Optional. Minimum time in minutes to run fan_only for pad drying. Defaults to 10 minutes.
- pad_dry_time_speed10: Optional. Minimum time in minutes to run fan_only for pad drying at speed 10. Defaults to 5 minutes.
- pad_drying_previous: Unix timestamp of when pad drying started. If 0, drying has not started.
- setpoint_temp
- indoor_temp
- max_fan_speed_entity
- min_fan_speed_entity

```
if (pad_drying_entity >= 100%) {
    if (cur_mode != 'off') {
        set evap_entity to off
    }
    exit script
}

if (cur_mode != 'fan_only') {
    if (cur_mode == off) {
        call set_evap_target: target_mode = fan_only, target_speed = 2, evap_entity = evap_entity, min_fan_speed_entity, max_fan_speed_entity
    } else {
        call set_evap_target: target_mode = fan_only, target_speed = -1, evap_entity = evap_entity, min_fan_speed_entity, max_fan_speed_entity
    }
}

if (pad_drying_previous == 0) {
    set pad_drying_entity to 0
} else {
    calculate:
        elapsed_minutes = (now() as seconds - pad_drying_previous) / 60
        current_fan_speed = current fan speed of evap_entity
        dry_time_minutes = pad_dry_time_speed1 + (current_fan_speed - 1) * (pad_dry_time_speed10 - pad_dry_time_speed1) / 9
        percent_dried_delta = (elapsed_minutes / dry_time_minutes) * 100
        new_percent_dried = min(100, percent_dried_delta + pad_drying_entity state)
    set pad_drying_entity to new_percent_dried
}

calculate:
   lower_fan = (setpoint_temp - indoor_temp) > 0 or pad_drying_entity >= 80

if (lower_fan) {
    calculate target_speed = max(1, current_fan_speed - 1)
    if target_speed != current_fan_speed {
        call set_evap_target: target_mode = fan_only, target_speed = target_speed, evap_entity = evap_entity, min_fan_speed_entity, max_fan_speed_entity
    }
}
```

## Set Evap Target Script

Inputs:
- evap_entity: The evaporative cooler mode select entity to control (e.g., select.roofevap1_e90248_evap_mode).
- fan_speed_entity: The evaporative cooler fan speed select entity to control (e.g., select.roofevap1_e90248_evap_fan_speed).
- target_mode: Target HVAC mode (off, fan_only, cool).
- target_speed: Target fan speed (1-10).
- min_fan_speed_entity: Optional. Entity defining minimum fan speed (1-10). If not provided min fan speed is 1
- max_fan_speed_entity: Optional. Entity defining maximum fan speed (1-10). If not provided max fan speed is 10

```
set current_mode = current state of evap_entity (mode select)
set current_fan_speed = current state of fan_speed_entity (fan speed select)

if (current_mode != target_mode) {
    if (target_mode == off and current_fan_speed >= 2) {
        target_speed = max(2, current_fan_speed - 2)
        if (current_mode != fan_only) {
            set target_mode to fan_only
        }
    }

    set evap_entity (mode select) to target_mode using select.select_option
    wait on evap_entity to reach target_mode (15 seconds timeout)
}

set current_fan_speed = current state of fan_speed_entity
if abs(target_speed - current_fan_speed) > 2 {
    if (target_speed > current_fan_speed) {
        target_speed = current_fan_speed + 2
    } else {
        target_speed = current_fan_speed - 2
    }
}

if (min_fan_speed_entity is defined) {
    set min_fan_speed = state of min_fan_speed_entity
    target_speed = max(target_speed, min_fan_speed)
}

if (max_fan_speed_entity is defined) {
    set max_fan_speed = state of max_fan_speed_entity
    target_speed = min(target_speed, max_fan_speed)
}

while (target_speed > 0 and current_fan_speed != target_speed && repeat.index < 15) {
    set fan_speed_entity to target_speed using select.select_option
    wait 2 seconds
    set current_fan_speed = current state of fan_speed_entity
}
```