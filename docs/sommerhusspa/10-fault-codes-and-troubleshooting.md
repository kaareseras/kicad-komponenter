# Fault Codes and Troubleshooting

## 1. Fault Philosophy

The Sommerhusspa controller distinguishes between **faults**, **warnings**, and **informational events**.

- **Faults** are conditions that require immediate protective action or that prevent the system from operating safely or correctly. Faults can shut down one or more outputs and may require manual intervention.
- **Warnings** are abnormal conditions that do not require an immediate shutdown but should be shown to the user and logged for diagnosis.
- **Informational events** record normal state changes, such as a Wi-Fi reconnect or a successful commissioning event.

### Fault Severity and Behaviour

#### Latching faults
Latching faults remain active until the triggering condition is cleared and the controller receives a manual reset, power cycle, or explicit fault-clear command. Use latching faults when a condition may indicate a hazardous state, a failed safety chain, or a potentially damaged actuator.

Typical latching faults in this system include:
- Heater safety-chain fault.
- Overtemperature fault.
- Valve runtime exceeded faults.
- Metering plausibility fault after heater shutdown verification fails.

#### Non-latching faults
Non-latching faults clear automatically when the underlying condition returns to normal. These should still be logged with start and end timestamps.

Typical non-latching faults in this system include:
- Low water detected.
- Temporary Wi-Fi disconnect.
- BLE commissioning timeout.

### Auto-retry vs Manual Reset

- **Auto-retry** is appropriate where the hardware can safely resume when the condition clears, for example after water level returns, Wi-Fi reconnects, or a temporary bus timeout recovers.
- **Manual reset** is required where the system needs human inspection before re-enabling power hardware, especially the heater, valves that exceeded maximum runtime, or outputs involved in an interlock violation.

### Recommended fault-handling principles

1. Always fail safe. When in doubt, de-energise heater, jet, fill, and drain outputs.
2. Low water always blocks heater and pumps.
3. Heater and jet are mutually exclusive; any violation should shut down the heater first, then the jet if necessary.
4. Cover sensor changes are logged, but they do not block heater or jet operation.
5. If circulation-related confirmation does not occur within 60 seconds of a heating sequence that expects it, raise a circulation timeout fault.
6. If the heater fault input changes while the heater is running, drop the heater immediately and verify the resulting power state using the metering ADC.
7. All faults should generate both a local OLED indication and a remote MQTT event.

## 2. Fault Code Table

The following code system is recommended:

- **F001-F009**: Water level and pressure-related faults.
- **F010-F019**: Temperature and thermal protection faults.
- **F020-F029**: Heater and safety-chain faults.
- **F030-F039**: Jet pump and interlock faults.
- **F040-F049**: Valve and flow-management faults.
- **F050-F059**: Circulation and process timeout faults.
- **F060-F069**: Metering and electrical plausibility faults.
- **F070-F079**: Bus and peripheral communication faults.
- **F080-F089**: Network, cloud, and remote-command faults.
- **F090-F099**: BLE commissioning and service faults.

### Fault Code Details

| Code | Name | Trigger condition | Immediate hardware response | Latching / auto-clear | Local display | MQTT reporting | Suggested troubleshooting |
|---|---|---|---|---|---|---|---|
| **F001** | Low Water Active | Low water safety input becomes active while system is idle or running. | Shut down heater, jet low, jet high, and circulation output. Block restart of heater and pumps until clear. | Auto-clearing after input is stable normal for 5 s. | OLED: `F001 LOW WATER` and water icon. | Publish `fault_code=F001`, `state=active`, `severity=critical`, `low_water=1`. | Check water level, skimmer/intake, sensor wiring, float switch mounting, and whether drain valve is leaking open. |
| **F002** | Pressure Sensor Out of Range | SM3041-015 reading is below valid minimum or above valid maximum for more than the debounce period. | Disable heater. Optionally stop circulation if pressure is used as a flow permissive. | Latching if used as a heater permissive; otherwise auto-clear when valid. | OLED: `F002 PRESSURE` with raw value page in diagnostics. | Publish `fault_code=F002`, `pressure_kpa`, `state=active`. | Verify 5 V or 3.3 V supply, sensor tubing, connector seating, ADC scaling, and damaged sensor diaphragm. |
| **F003** | Water Level Signal Unstable | Low water input chatters more than allowed within a defined interval. | Disable heater and pumps until the signal becomes stable. | Auto-clear after stable normal state for 30 s. | OLED: `F003 LEVEL NOISY`. | Publish `fault_code=F003`, `toggle_count`, `state=active`. | Inspect float switch bounce, loose terminals, EMI from relays, and missing pull-up or pull-down resistors. |
| **F010** | Water Temperature Sensor Missing | DS18B20 not detected on the 1-Wire bus during scan. | Disable heater. Keep non-heating functions available if safe. | Auto-clear after successful rediscovery and two valid samples. | OLED: `F010 TEMP SENSOR`. | Publish `fault_code=F010`, `sensor=ds18b20`, `state=active`. | Check data wire continuity, pull-up resistor, connector polarity, moisture ingress, and wrong GPIO assignment. |
| **F011** | Water Temperature Sensor CRC Error | Repeated DS18B20 CRC or read errors beyond retry threshold. | Disable heater. | Auto-clear after valid readings resume. | OLED: `F011 TEMP CRC`. | Publish `fault_code=F011`, `read_errors`, `state=active`. | Inspect cable length, grounding, 1-Wire pull-up strength, electrical noise near relay wiring, and solder joints. |
| **F012** | Overtemperature | Measured water temperature exceeds the safe high limit above the configured setpoint control band. | Immediately shut down heater. Keep circulation allowed if used for cool-down. | Latching; requires temperature to fall below reset threshold and manual reset. | OLED: `F012 OVER TEMP`. | Publish `fault_code=F012`, `temp_c`, `limit_c`, `state=active`. | Confirm sensor accuracy, relay state, heater command logic, welded relay contacts, and actual water flow. |
| **F013** | Temperature Control Deviation | Temperature cannot be held within ±0.5 °C of setpoint over the configured control window. | No forced shutdown by default; disable heater if deviation indicates runaway. | Warning-grade fault or configurable auto-clear fault. | OLED: `F013 TEMP DEV`. | Publish `fault_code=F013`, `temp_c`, `setpoint_c`, `deviation_c`. | Check calibration, circulation effectiveness, heater capacity, insulation, and sensor placement. |
| **F020** | Heater Safety Chain Open | Heater fault input indicates open safety chain when a heater request is present. | Do not energise heater relays. Maintain jet lockout while heater request is unresolved. | Latching until safety chain is healthy and manual reset is performed. | OLED: `F020 HEATER SAFE`. | Publish `fault_code=F020`, `heater_fault=1`, `state=active`. | Check thermal cut-out, flow switch if fitted, contactor auxiliary feedback, wiring loop continuity, and emergency interlock devices. |
| **F021** | Heater Fault During Run | Heater fault input changes state while heater is energised. | Drop all heater relays immediately, record metering snapshot, and inhibit further heating. | Latching. | OLED: `F021 HEATER TRIP`. | Publish `fault_code=F021`, `state=active`, `meter_verify=pending|pass|fail`. | Inspect safety-chain devices, relay coil wiring, contact bounce, and whether the heater assembly is overheating. |
| **F022** | Heater Power Present After Shutdown | After commanded heater shutdown, ADS131M08 indicates current or power above the off threshold for the verification window. | Force all heater outputs off, block jet, raise critical alarm. | Latching; manual reset only after inspection. | OLED: `F022 HEATER STUCK`. | Publish `fault_code=F022`, `power_w`, `current_a`, `state=active`. | Check welded relay or contactor contacts, PCB relay driver short, incorrect metering calibration, and miswired phases. |
| **F023** | Heater Metering Missing | Heater is commanded on, but no plausible heater current is detected after the allowed stabilisation time. | Shut down heater and log failed start. | Latching after repeated retries, otherwise auto-retry once. | OLED: `F023 NO HEAT POWER`. | Publish `fault_code=F023`, `power_w`, `expected=heating`. | Check relay outputs, fuse links, supply phases, contactor coil, ADC channels, and heater element continuity. |
| **F030** | Heater and Jet Interlock Violation | Logic detects simultaneous heater-enable and jet-enable command or overlapping output state. | Shut down heater immediately and then shut down jet if overlap persists. | Latching because it indicates a control integrity issue. | OLED: `F030 INTERLOCK`. | Publish `fault_code=F030`, `heater_cmd`, `jet_state`, `state=active`. | Review firmware state machine, output mapping, relay feedback, and race conditions around mode transitions. |
| **F031** | Jet Speed Transition Error | Jet command changes directly between low and high without the required 0.5 s dead-time, or feedback suggests overlap. | Turn off both jet outputs, enforce dead-time, and block restart until safe. | Latching after repeated occurrences; otherwise auto-clear after command reset. | OLED: `F031 JET TRANSITION`. | Publish `fault_code=F031`, `deadtime_ms`, `state=active`. | Check control logic, relay release time, mechanical contactor overlap, and output-expander timing. |
| **F032** | Jet Output Conflict | Low-speed and high-speed jet outputs are both active, or both output channels appear shorted together. | De-energise both jet outputs immediately. | Latching. | OLED: `F032 JET CONFLICT`. | Publish `fault_code=F032`, `jet_low=1`, `jet_high=1`. | Inspect wiring harness, interposing relays, PCB shorts, wrong expander bit mapping, and field wiring errors. |
| **F033** | Low Water Pump Block | A pump start request occurs while low water condition is active. | Reject pump start and keep pumps off. | Auto-clear when low water clears. | OLED: `F033 PUMP BLOCKED`. | Publish `fault_code=F033`, `reason=low_water`. | Restore water level, then verify input state and retry. If false, inspect low water sensor circuit. |
| **F040** | Fill Valve Runtime Exceeded | Fill valve remains energised continuously or cumulatively for more than 2 hours without successful completion. | De-energise fill valve and block further fill commands. | Latching; manual inspection recommended. | OLED: `F040 FILL TIMEOUT`. | Publish `fault_code=F040`, `runtime_s`, `limit_s=7200`. | Check supply water, valve coil, stuck-open mechanical valve, level sensing, plumbing restrictions, and control logic. |
| **F041** | Drain Valve Runtime Exceeded | Drain valve remains energised continuously or cumulatively for more than 1 hour. | De-energise drain valve and block further drain commands. | Latching. | OLED: `F041 DRAIN TIMEOUT`. | Publish `fault_code=F041`, `runtime_s`, `limit_s=3600`. | Inspect drain blockage, valve coil, stuck-open valve, plumbing kinks, and incorrect drain-complete detection. |
| **F042** | Fill and Drain Conflict | Fill and drain valves are commanded simultaneously. | Shut both valves immediately. | Latching because it indicates a logic or wiring fault. | OLED: `F042 VALVE CONFLICT`. | Publish `fault_code=F042`, `fill=1`, `drain=1`. | Review firmware sequencing, output mapping, expander writes, and any manual override commands. |
| **F050** | Circulation Start Timeout | A process requiring circulation does not achieve the expected circulation-related condition within 60 s. | Shut down heater request, keep circulation command off or retry per policy. | Latching for heater-related sequences; otherwise configurable auto-retry once. | OLED: `F050 CIRC TIMEOUT`. | Publish `fault_code=F050`, `timeout_s=60`, `state=active`. | Check circulation pump output, relay, plumbing blockage, air lock, pressure response, and whether the expected confirmation logic is correct. |
| **F051** | Heater Enable Without Circulation Confirmation | Heater is requested, but the circulation-related permissive is absent when expected. | Do not enable heater, or shut it down immediately if already started. | Auto-clear if the permissive arrives in time; latching if timeout occurs. | OLED: `F051 NO CIRC OK`. | Publish `fault_code=F051`, `circulation_ok=0`. | Confirm circulation signal source, pump status, pressure sensor behaviour, and software interlock timing. |
| **F060** | Metering ADC Communication Fault | ADS131M08 does not respond on SPI or returns invalid frames. | Shut down heater and any function relying on power verification. | Latching for heater operation; non-heating mode may continue with warning. | OLED: `F060 METER ADC`. | Publish `fault_code=F060`, `device=ads131m08`, `state=active`. | Check SPI wiring, chip select, clock integrity, reset line, power rails, and firmware driver initialisation. |
| **F061** | Metering Value Out of Range | Voltage, current, or power values exceed sane engineering limits or become internally inconsistent. | Shut down heater if readings affect safety verification. | Latching after confirmation over several samples. | OLED: `F061 METER RANGE`. | Publish `fault_code=F061`, `channel`, `value`, `state=active`. | Verify CTs or shunts, divider ratios, ADC calibration constants, loose neutral, and EMI coupling. |
| **F062** | Metering Plausibility Mismatch | Measured power does not match expected operating state, for example heater off but high power, or heater on with impossible values. | De-energise heater and log a critical diagnostic event. | Latching. | OLED: `F062 METER PLAUS`. | Publish `fault_code=F062`, `expected_state`, `measured_power_w`. | Check relay welding, wrong channel mapping, calibration, ADC synchronisation, and phase assignment. |
| **F070** | I2C Expander Not Responding | One of the three PCF8574 devices fails to acknowledge on I2C. | Freeze affected outputs in safe-off state where possible; shut down heater and jet if their control path is uncertain. | Latching until communication recovers and reset is performed. | OLED: `F070 I2C PCF8574`. | Publish `fault_code=F070`, `device`, `i2c_addr`, `state=active`. | Check addressing straps, SDA/SCL pull-ups, bus shorts, supply voltage, connector orientation, and address conflicts. |
| **F071** | I2C Lighting Driver Not Responding | PCA9635 does not acknowledge or configure correctly. | Lighting only is disabled; core spa operation continues. | Auto-clear after successful reinitialisation. | OLED: `F071 LIGHT I2C`. | Publish `fault_code=F071`, `device=pca9635`, `state=active`. | Check I2C address, power, bus loading, and whether another device is using the same address. |
| **F072** | Bus Stuck Low | I2C SDA or SCL remains low beyond the recovery threshold. | Disable dependent peripherals and attempt bus recovery sequence. | Auto-clear after successful recovery; escalate to latching after repeated failures. | OLED: `F072 BUS STUCK`. | Publish `fault_code=F072`, `bus=i2c`, `state=active`. | Inspect for solder bridges, damaged ICs, moisture, cable shorts, and devices holding the line low. |
| **F073** | SPI Peripheral Initialisation Fault | ADS131M08 initialises partially but does not enter the expected operating mode. | Disable metering-dependent features, block heater. | Latching. | OLED: `F073 SPI INIT`. | Publish `fault_code=F073`, `device=ads131m08`. | Confirm reset timing, oscillator or clock settings, firmware register writes, and board assembly quality. |
| **F080** | Wi-Fi Disconnected | Controller loses Wi-Fi connection for longer than the reporting grace period. | No hardware shutdown; local control continues. | Auto-clear when reconnected. | OLED: `F080 WIFI LOST`. | Publish retained `fault_code=F080`, `wifi=down`, plus reconnect event when restored. | Check SSID credentials, signal strength, AP uptime, antenna connection on the ESP32-S3-WROOM-1U-N8R8, and DHCP lease status. |
| **F081** | Cloud or MQTT Command Timeout | A remote command expects acknowledgement but times out. | Ignore the stale command and keep current safe state. | Auto-clear. | OLED: `F081 REMOTE TIMEOUT`. | Publish `fault_code=F081`, `command`, `timeout_ms`. | Verify broker connectivity, topic permissions, latency, client IDs, and whether the remote app is waiting on the correct acknowledgement topic. |
| **F082** | Remote Configuration Reject | Received remote configuration is invalid, incomplete, or outside allowed bounds. | Reject configuration, keep prior settings. | Auto-clear after next valid command. | OLED: `F082 CFG REJECT`. | Publish `fault_code=F082`, `field`, `reason`. | Check payload schema, units, range limits, and firmware version compatibility. |
| **F090** | BLE Commissioning Timeout | BLE setup session starts but does not complete within the allowed time. | End BLE commissioning mode; no effect on running spa outputs. | Auto-clear. | OLED: `F090 BLE TIMEOUT`. | Publish `fault_code=F090`, `state=active`, `session=expired`. | Retry pairing closer to the controller, check phone permissions, and verify BLE advertising is enabled. |
| **F091** | BLE Authentication Failed | Commissioning attempt uses invalid credentials or pairing token. | Reject provisioning command and close the session after repeated failures. | Auto-clear after cooldown; log as security event. | OLED: `F091 BLE AUTH`. | Publish `fault_code=F091`, `reason=auth_failed`. | Confirm provisioning PIN or token, app version, time validity, and whether the device was already provisioned. |
| **F092** | BLE Provisioning Data Invalid | Received SSID, password, cloud endpoint, or other provisioning data fails validation. | Do not apply new settings. | Auto-clear after valid retry. | OLED: `F092 BLE DATA`. | Publish `fault_code=F092`, `field`, `reason=invalid`. | Re-enter credentials, confirm UTF-8 handling, payload format, and maximum field lengths. |

## 3. Fault Reporting Requirements

Each fault record should include the following fields internally and over MQTT:

- `fault_code`: for example `F021`.
- `name`: human-readable name.
- `state`: `active`, `cleared`, or `latched`.
- `severity`: `warning`, `fault`, or `critical`.
- `timestamp`.
- `source`: input, sensor, bus, logic, or communication.
- `details`: compact JSON object with raw values.
- `requires_manual_reset`: boolean.

### Local display behaviour
On the OLED display, active faults should:
- Show the fault code on the first line.
- Show a short plain-English description on the second line.
- Alternate every few seconds between description and a basic action hint, such as `CHECK WATER LEVEL`.
- Show only the highest-severity active fault on the main screen.
- Allow scrolling through all active and recent faults in a diagnostics menu.

### Remote MQTT behaviour
When a fault becomes active, the controller should publish an immediate event and update retained state topics. A cleared fault should also generate an event.

Recommended payload example:

```json
{
  "fault_code": "F021",
  "name": "Heater Fault During Run",
  "state": "active",
  "severity": "critical",
  "timestamp": "2026-05-10T01:18:00+02:00",
  "requires_manual_reset": true,
  "details": {
    "heater_fault_input": 1,
    "heater_command": 1,
    "meter_verify": "pending"
  }
}
```

## 4. Warning-Level Events That Are Not Faults

The following conditions should be treated as warnings or informational events rather than faults.

### W001: Temperature approaching control limit
Trigger when water temperature enters a pre-warning band near the configured control threshold, for example within 0.2 °C of the stop-heating boundary.

- Local display: `TEMP NEAR LIMIT`.
- MQTT: publish warning event with current temperature and setpoint.
- Action: no shutdown. This is informational for tuning and diagnostics.

### W002: Wi-Fi reconnected
Trigger when Wi-Fi returns after a disconnect.

- Local display: optional brief banner `WIFI RESTORED`.
- MQTT: publish info event with reconnect timestamp, RSSI, and IP address.
- Action: no shutdown.

### W003: OTA update available
Trigger when the controller learns that a newer approved firmware version is available.

- Local display: `UPDATE AVAILABLE` in status rotation.
- MQTT: publish firmware current version and available version.
- Action: no shutdown. User can choose maintenance timing.

## 5. System Event Log

The controller should maintain a structured event log in non-volatile storage plus a rolling in-memory buffer.

### Events to log
Log at minimum:
- Fault active and fault cleared events.
- Warning and information events.
- Output state changes for heater, circulation, jet low, jet high, fill, drain, UV, and lighting.
- Safety input changes: low water, heater fault, and lid sensors.
- Setpoint changes and configuration updates.
- BLE commissioning start, success, failure, and timeout.
- Wi-Fi connect, disconnect, MQTT connect, MQTT disconnect.
- Boot reason, firmware version, factory reset, watchdog reset, brownout, and crash recovery.
- Metering anomalies and bus recovery attempts.

### Retention recommendation
- Keep the last **500 to 2,000 events** in flash, depending on wear budget and available storage.
- Keep the last **50 to 100 fault transitions** in a dedicated retained fault history.
- Include monotonic sequence numbers so remote tools can detect missed events.

### How to retrieve logs
#### Locally
- Provide a diagnostics menu on the OLED with recent fault codes and timestamps.
- Expose a serial or USB debug console for service use.
- Optionally allow log dump over a local service page.

#### Remotely
- Publish event records to an MQTT event topic.
- Provide a diagnostic command to fetch the last `N` events or the active-fault list.
- Expose firmware version, uptime, reset reason, RSSI, and bus health as retained telemetry.

## 6. Remote Diagnostics via MQTT

A practical MQTT topic layout is shown below. Replace `sommerhusspa/<device_id>` with the actual device path.

### Topics to subscribe to
- `sommerhusspa/<device_id>/telemetry/status`
- `sommerhusspa/<device_id>/telemetry/faults/active`
- `sommerhusspa/<device_id>/telemetry/faults/events`
- `sommerhusspa/<device_id>/telemetry/sensors/temperature`
- `sommerhusspa/<device_id>/telemetry/sensors/pressure`
- `sommerhusspa/<device_id>/telemetry/power/ads131m08`
- `sommerhusspa/<device_id>/telemetry/io/inputs`
- `sommerhusspa/<device_id>/telemetry/io/outputs`
- `sommerhusspa/<device_id>/telemetry/network`
- `sommerhusspa/<device_id>/telemetry/logs/recent`

### Diagnostic command topics
- `sommerhusspa/<device_id>/cmd/diag/get_active_faults`
- `sommerhusspa/<device_id>/cmd/diag/get_recent_events`
- `sommerhusspa/<device_id>/cmd/diag/get_bus_status`
- `sommerhusspa/<device_id>/cmd/diag/get_meter_snapshot`
- `sommerhusspa/<device_id>/cmd/diag/run_output_test`
- `sommerhusspa/<device_id>/cmd/diag/clear_latched_faults`
- `sommerhusspa/<device_id>/cmd/diag/reboot`

### Example diagnostic command payloads

Get recent events:

```json
{
  "limit": 50
}
```

Clear latched faults:

```json
{
  "ack": true,
  "reason": "Technician inspected heater relay"
}
```

Run output test:

```json
{
  "output": "circulation",
  "duration_ms": 3000,
  "allow_if_faulted": false
}
```

### Safety note for remote diagnostics
Remote output tests should be blocked when critical faults are active, especially low water, heater safety-chain, or unresolved interlock faults. Manual onsite inspection is preferable before clearing latching heater-related faults.

## 7. Display Error Codes on the OLED

The OLED should show compact but readable operator-facing codes.

### Main display format
- First line: `FAULT F021`
- Second line: short text, for example `HEATER TRIP`
- Third line, if available: action hint, for example `RESET AFTER CHECK`

### Suggested short display labels
- `F001 LOW WATER`
- `F010 TEMP SENSOR`
- `F012 OVER TEMP`
- `F020 HEATER SAFE`
- `F021 HEATER TRIP`
- `F022 HEATER STUCK`
- `F030 INTERLOCK`
- `F031 JET TRANSITION`
- `F040 FILL TIMEOUT`
- `F041 DRAIN TIMEOUT`
- `F050 CIRC TIMEOUT`
- `F060 METER ADC`
- `F070 I2C PCF8574`
- `F080 WIFI LOST`
- `F090 BLE TIMEOUT`

If multiple faults are active, the display should show the highest-severity code first and indicate the count, for example `+2 MORE`.

## 8. Physical Troubleshooting Guide

### Bad solder joint
Symptoms:
- Intermittent sensor readings.
- Bus devices disappearing when the board is tapped or warmed.
- Relays operating inconsistently.

How to diagnose:
- Perform visual inspection under magnification.
- Gently flex the PCB while monitoring telemetry.
- Reflow dull or cracked joints, especially on PCF8574, ADS131M08, connectors, and relay drivers.

### Expander address conflict
Symptoms:
- Two outputs change together unexpectedly.
- One PCF8574 is missing or mirrored.
- Fault F070 appears after boot.

How to diagnose:
- Confirm each PCF8574 address strap with a meter.
- Run an I2C bus scan in diagnostics.
- Verify the PCA9635 does not share an address with another device.

### Sensor wire break
Symptoms:
- DS18B20 missing or CRC errors.
- Pressure sensor reads fixed high or low.
- Low water input appears permanently active or inactive.

How to diagnose:
- Measure continuity end to end.
- Wiggle-test the harness while monitoring readings.
- Check for corrosion in connectors and moisture ingress.
- Confirm pull-ups and supply voltage at the sensor end.

### Relay welded closed
Symptoms:
- Heater current remains present after shutdown.
- Jet or valve continues running even though output command is off.
- Fault F022 or F062 occurs.

How to diagnose:
- Measure voltage across relay contacts when off.
- Compare commanded state with metered current.
- Inspect contactor or relay for overheating or arcing.
- Replace the relay and inspect snubber or load characteristics.

### EMI or grounding issues
Symptoms:
- Random DS18B20 CRC errors.
- I2C bus faults during relay switching.
- Spurious low water toggles.

How to diagnose:
- Inspect cable routing between logic and power wiring.
- Add or verify flyback suppression on coils.
- Check grounding strategy and shield termination.
- Capture events relative to relay actuation timestamps.

### Incorrect phase or metering wiring
Symptoms:
- Heater power appears on the wrong channel.
- Implausible readings from ADS131M08.
- False stuck-relay or no-power faults.

How to diagnose:
- Confirm phase-to-channel mapping.
- Verify CT or shunt polarity.
- Inject a known load and compare measured values.
- Recalibrate scaling constants.

## 9. Factory Reset Procedure and When to Use It

A factory reset should be used only when configuration corruption, failed commissioning, or unrecoverable communication settings prevent normal setup or service.

### Use factory reset when
- BLE or Wi-Fi provisioning data is invalid and cannot be corrected remotely.
- MQTT broker settings are wrong and the device cannot reconnect.
- Configuration schema changed across firmware versions and recovery is needed.
- A service technician is recommissioning a replacement controller.

### Do not use factory reset when
- A hardware fault is active, such as low water, heater safety-chain failure, welded relay, or sensor wiring issues.
- The issue can be resolved by clearing a latched fault after inspection.
- You still need preserved logs for troubleshooting, unless logs are exported first.

### Recommended factory reset procedure
1. Place the spa in a safe idle state. Ensure heater, jet, fill, and drain outputs are off.
2. If possible, export recent logs and active fault history over MQTT or service interface.
3. Press and hold the designated service button during power-up, or use a protected local maintenance menu action.
4. Keep the button held until the OLED shows `FACTORY RESET CONFIRM`.
5. Confirm the reset using a second deliberate action, such as a short press within 10 seconds.
6. Erase stored Wi-Fi, BLE commissioning, MQTT, calibration overrides, and user settings.
7. Preserve immutable device identity, hardware revision, and manufacturing data where applicable.
8. Reboot into commissioning mode and show `READY TO SETUP` on the OLED.

### After reset
- Recommission BLE and Wi-Fi.
- Restore setpoint and operating preferences.
- Verify sensor readings before enabling heater operation.
- Run a supervised output test for circulation, jet, valves, and heater permissives.
- Confirm no latching safety fault immediately reappears.

## Recommended Implementation Notes

To make this guide actionable in firmware, the controller should maintain a per-fault structure with:
- active state,
- first seen time,
- last seen time,
- clear debounce,
- retry counter,
- manual reset requirement,
- and a compact diagnostic payload.

This allows the system to distinguish between transient field noise and real equipment faults while still failing safe when the heater, pumps, or valves may be at risk.