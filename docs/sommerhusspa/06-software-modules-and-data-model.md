# 06 - Software Modules and Data Model

## Purpose

This document proposes a practical firmware module split and an initial internal data model for the Sommerhusspa controller.

It is not a final implementation spec, but a structure intended to make firmware work easier, safer, and more maintainable.

## Recommended software modules

### 1. board_support
Low-level hardware initialization and pin/bus ownership.

Responsibilities:
- initialize ESP32 GPIO
- initialize I2C bus
- initialize SPI bus
- initialize 1-Wire bus
- configure interrupt lines from expanders
- expose board-level constants and hardware IDs

### 2. io_expanders
Abstraction for the three PCF8574 devices.

Responsibilities:
- read input states from PCF8574 #0 and #2
- maintain shadow state for output bits
- write relay/LED outputs safely
- handle mixed-direction behavior on PCF8574 #2
- provide debounced logical button/input states to higher layers

### 3. lighting_service
Abstraction for PCA9635 lighting outputs.

Responsibilities:
- on/off light control output
- RGB PWM control
- predefined light states/effects if needed later

### 4. sensing_service
Sensor collection and normalization.

Responsibilities:
- read DS18B20 water temperature
- read water pressure sensor over I2C
- estimate level-related state from pressure if required
- validate raw sensor values and detect impossible readings

### 5. metering_service
Processing layer for ADS131M08 measurements.

Responsibilities:
- acquire raw ADC frames
- calculate per-phase RMS voltage/current
- calculate power values
- expose stable averaged measurements
- support command-verification hooks for heater and other loads

### 6. ui_service
Local user interface logic.

Responsibilities:
- handle external illuminated buttons
- handle onboard navigation buttons
- control button LEDs
- update OLED views
- manage menus and local status screens

### 7. control_service
Main operating logic and state machines.

Responsibilities:
- heater state machine
- jet state machine
- circulation/UV control
- fill/drain control
- temperature regulation
- command arbitration between local and remote requests

### 8. safety_supervisor
Cross-cutting safety and permissive logic.

Responsibilities:
- enforce heater/jet mutual exclusion
- enforce low-water block
- monitor heater fault input
- enforce valve runtime limits
- trigger fault states and safe shutdowns
- validate transitions using metering data where useful

### 9. connectivity_service
Remote communication and provisioning.

Responsibilities:
- Wi-Fi connectivity
- BLE commissioning flow
- OTA update handling
- remote telemetry publishing
- remote command intake

### 10. persistence_service
Storage of settings and retained runtime values.

Responsibilities:
- save configuration
- store calibration values
- store user setpoints
- persist selected counters or fault history

## Suggested internal data model

## Runtime state groups

### system_state
Global system-level fields.

Suggested fields:
- `boot_id`
- `uptime_s`
- `mode`
- `fault_active`
- `fault_code`
- `wifi_connected`
- `ble_commissioning_active`
- `ota_in_progress`

### sensor_state
Normalized sensor values.

Suggested fields:
- `water_temp_c`
- `pressure_raw`
- `pressure_converted`
- `level_ok`
- `heater_fault_input`
- `lid_1_active`
- `lid_2_active`

### button_state
User input state.

Suggested fields:
- `drain_pressed`
- `fill_pressed`
- `light_pressed`
- `jet_pressed`
- `nav_up_pressed`
- `nav_down_pressed`
- `nav_left_pressed`
- `nav_right_pressed`
- `nav_enter_pressed`

### output_state
Commanded outputs.

Suggested fields:
- `heater_l1_cmd`
- `heater_l2_cmd`
- `heater_l3_cmd`
- `jet_low_cmd`
- `jet_high_cmd`
- `circulation_uv_cmd`
- `valve_fill_cmd`
- `valve_drain_cmd`
- `button_led_drain`
- `button_led_fill`
- `button_led_light`
- `button_led_jet`
- `light_onoff_cmd`
- `light_rgb_r`
- `light_rgb_g`
- `light_rgb_b`

### control_state
High-level logical states.

Suggested fields:
- `heater_state`
- `jet_state`
- `fill_state`
- `drain_state`
- `temperature_control_state`
- `requested_setpoint_c`
- `effective_heater_permission`
- `low_water_block`
- `jet_heater_interlock`

### timing_state
Timers and timeouts.

Suggested fields:
- `jet_transition_deadtime_ms`
- `fill_runtime_s`
- `drain_runtime_s`
- `heater_enable_wait_s`
- `fault_latched_at_ms`

### metering_state
Processed electrical data.

Suggested fields:
- `voltage_l1_rms`
- `voltage_l2_rms`
- `voltage_l3_rms`
- `current_l1_rms`
- `current_l2_rms`
- `current_l3_rms`
- `power_l1`
- `power_l2`
- `power_l3`
- `power_total`
- `heater_current_verified`

## Event-driven recommendations

The firmware should prefer event-driven updates where practical.

Useful event types:
- button pressed/released
- expander interrupt
- heater fault changed
- low-water state changed
- timeout expired
- remote command received
- metering verification failed

## State ownership principle

Each field should have one clear owner.

Recommended pattern:
- sensing modules own raw and normalized sensor values
- control modules own requested operating states
- output modules own final hardware writes
- safety supervisor owns fault latches and hard blocks

This avoids unclear interactions and makes debugging much easier.

## Logging recommendations

Useful structured log categories:
- `BOOT`
- `IO`
- `SENSOR`
- `CONTROL`
- `SAFETY`
- `METERING`
- `UI`
- `NET`
- `OTA`

## Future refinement

Likely next documentation steps:
- define concrete state-machine diagrams
- define fault codes and severity classes
- define remote API/telemetry schema
- define commissioning flow in detail
