# 05 - Firmware Architecture Notes

## Purpose

This file captures the current architectural implications for firmware development based on the confirmed hardware and behavior mapping.

## System model

The Sommerhusspa controller should be treated as a connected embedded control system with both local autonomy and remote-service functionality.

### Product characteristics
- local standalone control must exist
- Wi-Fi is used for remote control and remote management
- BLE is intended for commissioning only
- OTA updates are required
- USB exists but is not part of normal operation

## Recommended firmware subsystems

### 1. Hardware abstraction layer
Provide ownership and access layers for:
- I2C devices
- SPI metering ADC
- 1-Wire temperature sensing
- direct GPIO inputs
- expander-backed inputs/outputs

### 2. I/O service layer
Suggested responsibilities:
- PCF8574 shadow-state handling
- interrupt-driven expander input refresh
- output state management
- button LED state handling
- lighting control via PCA9635

### 3. Control state machines
Recommended state machines:
- heater state machine
- jet state machine
- fill/drain valve state machine
- circulation/UV state machine
- temperature regulation state machine

### 4. Safety supervisor
Should monitor:
- heater fault input
- low-water state
- interlock violations
- timeout conditions
- metering-confirmed shutdown / operation behavior

### 5. Metering subsystem
ADS131M08 processing should support:
- per-phase RMS voltage
- per-phase RMS current
- per-phase power
- total power
- accumulated energy
- component-operation verification logic

## Important implementation notes

### PCF8574 mixed-direction handling
PCF8574 #2 mixes button inputs and button LED outputs. Firmware should not treat it as a simple pure-input or pure-output device.

Recommended approach:
- maintain a software shadow register
- write full intended port state when changing LED bits
- read back/poll inputs without losing output state assumptions

### Command vs verification
Some outputs should not only be commanded; they should also be behaviorally verified where possible.

Examples:
- heater command should later be cross-checked against metering current draw
- shutdown on heater fault should be verified through power/current decay
- future diagnostics may infer pump/load behavior from phase measurements

### Separation of concerns
Keep these concepts separate in firmware:
- command state
- permissive/interlock state
- physical feedback state
- inferred runtime verification from metering

## Connectivity architecture

### Wi-Fi
Use for:
- remote control
- remote management
- reporting and diagnostics
- OTA update transport

### BLE
Use for:
- provisioning
- commissioning workflows
- possibly first-time setup only

## Documentation status

At this stage, the hardware and major control rules are sufficiently defined to begin firmware planning. Remaining work should focus on:
- exact state-machine definitions
- fault classification strategy
- thresholds and filtering
- remote API/data model
- commissioning flow details
