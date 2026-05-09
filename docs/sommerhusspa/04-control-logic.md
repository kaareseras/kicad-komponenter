# 04 - Control Logic

## Overview

This document captures the currently confirmed control rules and operating constraints for the Sommerhusspa controller.

## Core interlocks

### Heater vs jet pump
- Heater and jet pump must **not** run at the same time.
- If any heater phase output is active, jet operation is blocked.
- If jet operation is active, heater operation is blocked.

### Jet speed exclusivity
- Jet Low and Jet High are mutually exclusive.
- Required dead-time between Low and High transitions: **0.5 s**.
- User control cycle for the jet button:
  - Off -> Low -> High -> Off

### Fill vs drain
- Fill and Drain are mutually exclusive.
- Fill valve and Drain valve are spring-return and close when de-energized.

## Heater enable conditions

The heater may run only when all relevant conditions are satisfied.

Confirmed requirements:
- circulation state must be active
- heater fault must be inactive / okay
- low-water condition must not be active
- jet pump must be off

### Heater start timing
- Heater may enable **immediately** when circulation state becomes active.
- If the expected circulation/heater-permission-related condition does not become valid within **1 minute**, this should be treated as a timeout fault.

### Heater fault reaction
- If heater fault changes state while the heater is running:
  - shut down the heater
  - verify the shutdown using measured current draw

## Water-level related behavior
- Low water level blocks **both heater and pumps**.
- Pressure sensor is used to estimate water level and stop filling.

## Cover-sensor effect
- Cover sensors do **not** affect heater or jet operation.

## Valve runtime limits
- **Fill maximum runtime:** 2 hours
- **Drain maximum runtime:** 1 hour

These are important protective bounds and should be implemented as explicit timeout logic.

## Temperature control behavior
- Normal operation should hold the water temperature at the configured setpoint within **+/-0.5 °C**.

## Metering-assisted verification

The energy metering subsystem is not only for telemetry. It is also intended for functional verification.

Expected firmware uses include:
- verifying that commanded components are actually running properly
- supporting future energy-management functions
- confirming heater shutdown/current behavior after a fault event

## Notes for later refinement

The following implementation details may still need further refinement in firmware design:
- exact definition of the 1-minute heater/circulation timeout condition
- exact current-draw thresholds used for verification
- debounce/filter policy for fault and level inputs
- whether there are additional state-machine rules around fill/drain transitions
