# Commissioning and Calibration Guide

This guide describes how to commission, calibrate, verify, and hand over a Sommerhusspa control system based on the following hardware:

- ESP32-S3-WROOM-1U-N8R8 main controller.
- 3 x PCF8574 I/O expanders.
- PCA9635 LED PWM controller for lighting.
- ADS131M08IPBSR SPI energy metering front end.
- DS18B20 water temperature sensor.
- SM3041-015 pressure sensor.
- HS96L03W2C03 OLED display.
- 3-phase heater output with one 230 VAC relay per phase.
- Jet pump with low-speed and high-speed control.
- Circulation and UV output.
- Fill and drain valves with 12 V spring-return actuators.
- Four illuminated user buttons.
- Five onboard navigation buttons.
- Level sensor, two lid sensors, and heater fault input.
- BLE used for commissioning only.

> Warning: Commissioning includes live mains measurements and tests on high-power loads. Only qualified personnel should perform work on the heater relays, jet pump outputs, and any mains-connected wiring. Isolate power before rewiring. Use appropriate PPE, fused test leads, and an RCD-protected supply.

## 1. Overview of the commissioning workflow

Use the following sequence for a predictable and repeatable commissioning process:

1. Perform a visual inspection of the PCB, wiring, connectors, and enclosure.
2. Power the controller board without external loads connected.
3. Confirm successful boot, display operation, button response, and BLE commissioning access.
4. Run the factory reset or first-boot flow.
5. Read system identity information and verify all expected peripherals are detected.
6. Set date, time, and timezone.
7. Calibrate the temperature sensor.
8. Calibrate the pressure sensor and define empty and full thresholds.
9. Calibrate the energy metering channels for voltage and current.
10. Commission outputs one by one with loads disconnected where possible.
11. Verify lighting channels and button illumination.
12. Verify all safety inputs.
13. Provision Wi-Fi and configure the remote MQTT endpoint.
14. Update to the approved firmware release using OTA.
15. Perform an integrated system test under real load.
16. Save commissioning records and calibration values.
17. Complete the customer hand-over checklist.

Do not skip ahead to full-load testing before all sensor calibration and safety checks have passed.

## 2. Initial board power-on without loads

Before connecting pumps, valves, heater contact loads, or other field wiring, perform an unloaded power-on check.

### Preparation

- Inspect for shipping damage, bent pins, loose terminals, solder bridges, or misplaced jumpers.
- Verify the correct ESP32-S3 module, ADC, I/O expanders, lighting driver, and display are fitted.
- Confirm that mains outputs are not connected to the heater, pump, or circulation loads during the first boot.
- If the design includes pluggable terminals, confirm each connector is seated correctly and labelled.
- Verify the DS18B20 and pressure sensor connections are present if they are required for boot diagnostics.

### First power application

1. Apply the board supply only.
2. Confirm that current draw is within the expected unloaded range for the control electronics.
3. Check that the ESP32 boots without repeated reset cycles.
4. Verify the OLED display initializes and shows either the boot screen, diagnostics page, or commissioning prompt.
5. Test the five onboard navigation buttons.
6. Confirm BLE commissioning mode is advertised if first-boot logic expects it.
7. Check that no relay energizes unexpectedly.

### Accept criteria

- Stable boot within the normal startup time.
- No overheating components.
- No unexplained reboot loop.
- Display readable and responsive.
- Input expansion and lighting devices detectable on the bus.

If the board fails at this stage, stop and resolve hardware issues before connecting any external load.

## 3. Factory reset / first-boot procedure

A known baseline is essential before calibration.

### When to use factory reset

Perform a factory reset when:

- The controller has unknown settings.
- A board has been reworked or replaced.
- Calibration data may belong to another spa installation.
- A failed commissioning attempt needs to be restarted cleanly.

### Recommended procedure

1. Enter the service or installer menu from the display, or connect through the BLE commissioning interface.
2. Select **Factory Reset** or **Reset Commissioning Data**.
3. Confirm that network credentials, MQTT settings, calibration coefficients, and user schedules will be erased.
4. Reboot the controller.
5. Verify that the first-boot wizard starts.

### After reset, verify that default values are restored

- Device name or serial placeholder is reset.
- Wi-Fi credentials are cleared.
- MQTT endpoint is empty or set to factory default.
- Sensor calibration values are returned to default offsets and gains.
- Output timing values return to safe defaults.
- OTA channel is set to the approved commissioning channel.

If the firmware supports exporting settings before reset, save the original configuration first for traceability.

## 4. System identification

Before calibration, verify that the controller can identify itself and all major peripherals.

### Read and record hardware identity

Record the following where available:

- Product name and controller type.
- Firmware version and build date.
- Hardware revision.
- ESP32 MAC address or unique chip ID.
- Board serial number.
- Commissioning technician name and date.

### Verify detected peripherals

Check that the firmware reports the presence of:

- Three PCF8574 expanders.
- One PCA9635 LED driver.
- One ADS131M08 energy metering IC.
- One DS18B20 temperature sensor.
- One pressure sensor input.
- OLED display interface.

### Verifying PCF8574 addresses

The exact address map depends on the PCB strapping. During commissioning:

1. Open the diagnostics or I2C scan page.
2. Confirm that exactly three PCF8574 devices are detected.
3. Compare detected addresses to the schematic or manufacturing BOM.
4. Activate a known input or output linked to each expander and confirm the correct device changes state.
5. If any address is missing or duplicated, inspect address strap resistors, soldering, and bus wiring.

Also verify that the PCA9635 address appears as expected and does not conflict with any PCF8574 address.

### Recommended record

Document a table with:

- Device type.
- Bus address.
- Detected status.
- Functional check result.

## 5. Date/time configuration

Accurate timekeeping is required for logs, schedules, OTA audit records, and remote diagnostics.

### Procedure

1. Set timezone first.
2. Set date and local time manually through the display or BLE commissioning interface, or allow NTP synchronization once Wi-Fi is available.
3. If the firmware supports NTP, configure the preferred time source.
4. Reboot or resync and confirm the time persists.

### Verification

- System logs show the correct local timestamp.
- Scheduled functions use the correct time basis.
- Remote MQTT telemetry timestamps are correct.

If the controller does not contain a battery-backed RTC, note that time may be lost until Wi-Fi and NTP are restored after a power outage.

## 6. Temperature sensor calibration

The DS18B20 is used for water temperature monitoring. This sensor is factory-calibrated, but installation error and sensor placement can introduce offset.

### Expected accuracy

Typical DS18B20 accuracy is:

- Approximately ±0.5 °C over the common operating range around room and spa temperatures.
- Slightly worse at the extremes of the device range.

In practical spa operation, total installed accuracy also depends on:

- Sensor immersion depth.
- Thermal contact to flowing water.
- Cable length and noise.
- Conversion timing and filtering.

A realistic installed target after calibration is usually within ±0.3 °C to ±0.5 °C against a trustworthy reference thermometer.

### Reference setup

Use:

- A calibrated reference thermometer with known accuracy better than the required system accuracy.
- A stable water bath or circulating water condition.
- Enough settling time for both sensors to stabilize.

### How to read the DS18B20 calibration offset

The firmware should expose at least:

- Raw DS18B20 reading.
- Applied offset value.
- Final compensated temperature value.

During commissioning:

1. Place the DS18B20 and the reference thermometer in the same well-mixed water.
2. Wait for thermal stabilization.
3. Read the raw DS18B20 value.
4. Compute the offset:

   ```text
   offset = reference_temperature - raw_ds18b20_temperature
   ```

5. Enter that offset in the commissioning interface.
6. Save settings.
7. Re-read the compensated value and confirm it matches the reference within the acceptance band.

### Good practice

- Calibrate near the normal operating water temperature rather than at room temperature only.
- Repeat the check at a second temperature point if required by the specification.
- If the offset required is unexpectedly large, inspect sensor placement and wiring before accepting the value.

## 7. Pressure sensor calibration

The SM3041-015 pressure sensor is used to infer water level from hydrostatic pressure. Calibration must convert electrical output into meaningful water level thresholds.

### General principle

Hydrostatic pressure increases with water depth. The firmware should convert the raw sensor signal into engineering units, then into estimated water column height.

A generic conversion model is:

```text
pressure_engineering = (raw_counts - offset_counts) × scale
water_level = (pressure_engineering - zero_pressure) / fluid_density_factor
```

The exact form depends on the analogue front end, ADC scaling, sensor transfer function, mounting height, and any compensation implemented in firmware.

### Converting raw pressure to water level

To derive a usable conversion:

1. Record the raw pressure value with the spa empty or at the defined zero reference level.
2. Fill to a known reference height.
3. Record the new raw pressure value.
4. Calculate counts-per-millimetre or pressure-per-millimetre.
5. Enter the resulting offset and gain into the firmware.

If the firmware uses direct empty and full calibration points rather than explicit gain and offset, store those two points and let the firmware interpolate.

### Calibrating empty and full thresholds

Define at least these thresholds:

- **Empty threshold**: below this level, heating and pump operation that could cause damage must be blocked.
- **Minimum safe operating level**: level required for circulation and heater enable.
- **Full threshold**: level considered nominally full.
- **Overfill threshold** if supported.

Recommended procedure:

1. Drain the spa to the defined empty calibration point.
2. Record the raw sensor value and save it as **empty reference**.
3. Fill to the normal full level using a ruler, sight mark, or mechanical datum.
4. Record the raw sensor value and save it as **full reference**.
5. Set the minimum safe operating threshold between empty and full according to the hydraulic design.
6. Test hysteresis so the system does not chatter around the threshold.

### Verification

- Fill and drain slowly through the threshold region.
- Confirm displayed level changes smoothly.
- Confirm pump and heater interlocks trigger at the intended points.
- Confirm no false trips from noise or sloshing beyond the allowed debounce limits.

## 8. Energy metering calibration

The ADS131M08IPBSR measures three-phase voltage and current channels. Accurate calibration is important for energy reporting, power limiting, and diagnostics.

> Warning: This section requires work on live mains measurement circuits. Use isolated instruments rated for the installation category. If you are not qualified for mains metering calibration, do not proceed.

### Preparation

- Use a calibrated true-RMS meter or power analyser as the reference.
- Ensure correct CT orientation or current sensing polarity on all phases.
- Verify voltage measurement scaling components match the design.
- Confirm the ADC sampling configuration is set to the approved firmware defaults.

### Voltage channel calibration

Each phase should support independent offset and gain calibration.

#### Offset calibration per phase

1. De-energize the measured voltage input if the design and procedure allow a safe zero-input check.
2. Read the raw voltage channel value for L1, L2, and L3.
3. Store the zero reference or offset correction for each phase.
4. Reapply power and confirm the channels return to normal readings.

If a true zero-input condition is not practical, use the firmware's factory offset routine and verify that residual no-load reading is acceptably small.

#### Gain calibration per phase

1. Apply a known and stable mains voltage.
2. Measure actual RMS voltage on L1, L2, and L3 with the reference meter.
3. Read the controller's reported voltage for each phase.
4. Calculate phase gain:

   ```text
   gain = reference_voltage / measured_voltage_by_controller
   ```

5. Apply the new gain coefficient for each phase.
6. Recheck all three phases.

### Current channel calibration

Each current channel should also support independent calibration.

#### Offset calibration

1. Ensure no load current is flowing through the channel under test.
2. Read the raw or compensated current value.
3. Trim offset until the displayed no-load current is at or near zero within the specified residual error.

#### Gain calibration

1. Apply a known stable load on one phase at a time, or use a balanced known load.
2. Measure actual RMS current with a calibrated clamp meter or power analyser.
3. Read the controller value for each current channel.
4. Calculate and apply the gain coefficient for each phase.
5. Re-test at a second current point if possible.

### Power and energy verification

After voltage and current calibration:

1. Apply a known resistive load.
2. Compare reported real power per phase to the reference power analyser.
3. Verify that apparent power and power factor are plausible.
4. Run the load for a known period, for example 10 or 30 minutes.
5. Compare accumulated energy to the expected result:

   ```text
   energy = average_power × time
   ```

### Acceptance guidance

Use the project specification if defined. If not, a practical target for a correctly built control system is:

- Voltage error within approximately 1 percent after calibration.
- Current error within approximately 1 to 2 percent, depending on sensor type and low-current operating point.
- Power error consistent with combined voltage, current, and phase errors.
- Energy accumulation error small enough for operational monitoring and customer reporting.

If one phase reads incorrectly, check phase mapping, CT polarity, channel assignment, and neutral reference assumptions before adjusting coefficients further.

## 9. Output commissioning

Commission every output individually before running the full spa.

### General rules

- Label each field wire before testing.
- Keep hazardous loads disconnected until logic behaviour is confirmed.
- Use a meter or test lamp where possible before energizing the actual actuator.
- Confirm fail-safe behaviour after power loss.

### Heater relay test

> Caution: The heater outputs switch 230 VAC on three separate phases. Treat all heater wiring as hazardous.

Recommended procedure:

1. Verify that the heater is physically disconnected or isolated for the initial logic test.
2. Command each heater phase relay on and off individually through the service menu.
3. Confirm the correct relay indicator and output state.
4. Measure continuity or output voltage safely to verify each relay contact.
5. Once basic operation is confirmed, connect the heater load.
6. Command heat demand only when water level, flow conditions, and safety inputs are valid.
7. Confirm that all required phases energize only under valid heating conditions.
8. Confirm that a heater fault, low level, or open lid condition blocks heating if required by design.

### Jet pump speed switching test

The low-speed and high-speed outputs must never create an illegal simultaneous drive condition.

Procedure:

1. Test low-speed command only.
2. Confirm the correct relay or output energizes.
3. Test high-speed command only.
4. Confirm interlocking logic drops low speed before high speed engages, if required by the motor scheme.
5. Observe current draw and motor behaviour during transition.
6. Confirm that stopping the pump removes both speed commands.

### Valve travel time measurement

The fill and drain valves use 12 V spring-return actuation. Travel timing affects fill logic and alarms.

Procedure:

1. Command the fill valve open and measure the time from energization to confirmed flow or full travel.
2. De-energize and measure spring-return closure time.
3. Repeat for the drain valve.
4. Enter the measured opening and closing times if the firmware uses them for sequencing or fault detection.
5. Confirm the controller does not overdrive the valve longer than necessary.

### Circulation/UV test

1. Command the circulation or UV output on.
2. Verify the correct relay or driver state.
3. Confirm the connected load energizes.
4. Measure current draw if specified.
5. Command off and verify clean de-energization.

### Illuminated button check

Verify both the switch input and illumination output for each of the four illuminated buttons.

## 10. Lighting commissioning

The PCA9635 provides PWM-based lighting control. Commissioning should verify channel mapping, brightness linearity, and acceptable visual behaviour.

### Procedure

1. Identify each lighting channel and its connected load.
2. Set each channel to 0 percent, 25 percent, 50 percent, 75 percent, and 100 percent duty cycle.
3. Confirm the correct fixture responds.
4. Check for flicker, instability, or uneven brightness.
5. Verify that the minimum non-zero PWM value produces visible light if that is desired.
6. Verify that 0 percent is fully off.
7. If grouped scenes are supported, test each scene preset.

### PWM calibration considerations

- If LED current is fixed in hardware, commissioning mainly verifies channel mapping and useful PWM range.
- If firmware allows per-channel brightness scaling, use it to balance fixtures with visibly different brightness.
- Avoid setting minimum duty cycles so low that the lights flicker visibly.

Record any per-channel brightness trims used during commissioning.

## 11. Safety input verification

All safety-related inputs must be tested functionally, not just observed on a diagnostics page.

### Level sensor

1. Simulate or create a low-level condition.
2. Confirm the controller flags the low-level state.
3. Verify that prohibited outputs, especially heating, are blocked.
4. Restore normal level and confirm the state clears with the intended debounce and hysteresis.

### Lid sensors

1. Test each lid sensor independently.
2. Confirm the UI reports lid state correctly.
3. Verify any configured behaviour, such as disabling jets, limiting heat, or issuing a warning.
4. Confirm that wiring faults do not appear as a valid safe state unless intentionally designed that way.

### Heater fault input

1. Simulate the heater fault input.
2. Confirm the heater output drops immediately or within the required response time.
3. Verify the fault is logged with timestamp.
4. Confirm the fault reset behaviour matches the specification, whether automatic, latched, or service-reset only.

### Input logging

For each safety input, record:

- Normal state.
- Alarm state.
- Whether the logic is active-high or active-low.
- The protective action taken by the firmware.

## 12. Wi-Fi provisioning to the customer's network

Provision Wi-Fi only after local commissioning is complete enough that troubleshooting can still be done offline if needed.

### Procedure

1. Open the network setup page on the display or through BLE commissioning.
2. Scan for available networks.
3. Select the customer's SSID.
4. Enter the password carefully.
5. Save and connect.
6. Confirm that the controller receives an IP address.
7. Verify internet reachability if required for MQTT and OTA.

### Good practice

- Prefer the final installation network, not a temporary phone hotspot, unless explicitly needed for staging.
- Record the SSID used, but do not store the password in plain text in commissioning notes unless your process explicitly permits it.
- Confirm reconnection after a power cycle.

If the site Wi-Fi is weak at the spa location, record the RSSI and raise the issue before hand-over.

## 13. Remote endpoint configuration

Configure the remote endpoint only after network connectivity is stable.

### MQTT settings to configure

At minimum, enter and verify:

- Broker URL or hostname.
- Port number.
- TLS enabled or disabled according to deployment policy.
- Username.
- Password or token.
- Client ID.
- Device topic root.
- Telemetry publish interval if configurable.

### Verification

1. Save the settings.
2. Force a reconnect.
3. Confirm successful broker connection on the diagnostics page.
4. Verify the device publishes telemetry.
5. Verify the device receives any required control or configuration topics.

### Security note

Use unique credentials per installation where possible. Do not leave factory-default broker credentials active in the delivered system.

## 14. OTA update to latest firmware

Do not hand over a newly commissioned system on outdated firmware unless there is a documented reason.

### Procedure

1. Confirm the current firmware version.
2. Compare it to the approved release for this hardware revision.
3. Initiate OTA update from the maintenance menu or remote service interface.
4. Monitor download, verification, and reboot.
5. After reboot, confirm the new version is active.
6. Recheck calibration persistence and critical I/O state.

### After OTA, verify

- No calibration data was lost.
- Wi-Fi and MQTT settings were retained as intended.
- Display, buttons, sensor readings, and outputs still function.
- Build version and release channel are recorded.

If the firmware includes rollback support, confirm the rollback state is healthy after a successful boot.

## 15. Initial system test under load

This is the final integrated validation before hand-over.

### Fill test

1. Start from a known low or empty level.
2. Command fill.
3. Confirm the fill valve opens and water level rises as expected.
4. Confirm the controller stops filling at the intended target.
5. Check for leaks, overshoot, and stable pressure-derived level reading.

### Heat test

1. Confirm the spa is at a safe operating level.
2. Enable heating with all required safety conditions satisfied.
3. Verify that the heater relays engage correctly.
4. Observe energy metering values and current draw.
5. Confirm water temperature rises plausibly over time.
6. Confirm the heater cycles off appropriately when the setpoint is reached or when an interlock opens.

### Jet test

1. Run the jet pump at low speed.
2. Verify stable operation.
3. Switch to high speed.
4. Confirm correct transition and expected hydraulic behaviour.
5. Confirm no nuisance trips on level or fault inputs.

### Combined behaviour

Where the product specification allows, test realistic combined operation of circulation, heating, lighting, and user inputs. Confirm the controller remains stable and responsive.

## 16. Saving commissioning data

Commissioning data is valuable for support, warranty analysis, and future service.

### What to log

Record at minimum:

- Device serial number and hardware revision.
- Firmware version before and after OTA.
- Date, time, and technician name.
- ESP32 unique ID or MAC address.
- Detected I2C device addresses.
- Temperature offset.
- Pressure empty and full calibration values.
- Pressure thresholds and hysteresis values.
- Voltage offset and gain per phase.
- Current offset and gain per phase.
- Output test results.
- Safety input test results.
- Wi-Fi SSID.
- MQTT broker URL and client ID.
- Any deviations, faults, or rework performed.

### Where to save it

Use the project's approved storage locations, for example:

- Non-volatile commissioning record on the controller.
- Service laptop commissioning file.
- Manufacturing or service database.
- Installation report PDF stored with the customer job record.

Do not rely on memory alone. Save the final values in a place that can be retrieved during future service.

## 17. Checklist for hand-over to customer

Before hand-over, confirm all of the following:

- Controller boots normally without service tools attached.
- Date, time, and timezone are correct.
- Temperature reading is calibrated and plausible.
- Water level reading is calibrated and stable.
- Energy metering channels are calibrated and verified.
- Heater outputs operate correctly and safely.
- Jet pump low and high speeds operate correctly.
- Circulation and UV output operate correctly.
- Fill and drain valves operate correctly.
- Lighting channels and illuminated buttons operate correctly.
- Level sensor, lid sensors, and heater fault input have been tested.
- Wi-Fi is connected to the customer network.
- MQTT connection is online and telemetry is visible.
- OTA update has been completed or the approved firmware version is installed.
- Calibration and commissioning records have been saved.
- Customer-facing controls and indicators have been explained.
- Any installation limitations or open issues have been documented.

A hand-over is only complete when the system is safe, documented, networked as required, and reproducibly operating under real conditions.
