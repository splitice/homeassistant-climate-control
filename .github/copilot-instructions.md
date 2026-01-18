# Copilot Instructions

This project implements a climate control algorithm for Home Assistant that automates an evaporative cooler.

## Key Features
- Maintains target temperature using persistent control loop
- Handles state transitions (off → fan_only → cool)
- Adjusts fan speed (1-5) based on temperature difference
- Implements time-based thresholds to prevent rapid cycling
- Snapshot/restore for safe automation exit

## Technology
- Home Assistant YAML automation
- Jinja2 templating for state logic
- Entity types: climate (evap), sensor (temperature), input_boolean (control), input_number (setpoint)

## Requirements
17 numbered requirements (R1-R17) documented in README.md map to specific code sections in evap-control.automation.
