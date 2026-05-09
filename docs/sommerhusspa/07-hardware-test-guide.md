# Sommerhusspa Hardware Bring-Up and Test Guide

## 1. Overview and prerequisites

This document is a practical bring-up and subsystem validation procedure for a Sommerhusspa controller board based on the ESP32-S3-WROOM-1U-N8R8. It is intended for a single developer validating a newly assembled board on the bench before full system integration.

Work through the steps in order. Do not connect mains-powered loads until the low-voltage sections have been verified. Record measured values as you go. If a step fails, stop and resolve the fault before continuing to the next subsystem.

### Main hardware under test

- ESP32-S3-WROOM-1U-N8R8 main MCU
- 3 x PCF8574 I/O expanders at I2C addresses 0x40, 0x42, and 0x44
- PCA9635 LED driver on I2C, expected address approximately 0x65
- ADS131M08IPBSR 8-channel energy metering ADC on SPI
- DS18B20 1-Wire water temperature sensor on GPIO38
- HS96L03W2C03 OLED display on I2C
- SM3041-015-D-C-3-S pressure sensor on I2C
- Relay and actuator outputs for heater, jet pump, circulation/UV, fill valve, and drain valve
- User inputs and safety inputs, including heater fault on GPIO48
- Interrupt lines from PCF8574 #0 on GPIO8 and PCF8574 #2 on GPIO9

### Required tools

- Bench power supply if the board can be powered externally
- USB cable suitable for data, not charge-only
- Host PC with ESP32 flashing tools and a serial terminal
- Digital multimeter
- Oscilloscope with at least 2 channels recommended
- Logic analyser recommended for I2C, SPI, and 1-Wire troubleshooting
- Insulation-safe test leads and grabbers
- Known-good Wi-Fi network for connectivity test
- Optional: current-limited lamp or fused mains test fixture for relay output validation

### Recommended software

- `esptool.py` or `idf.py`
- Serial terminal such as `minicom`, `screen`, PuTTY, or the ESP-IDF monitor
- Firmware image containing dedicated hardware test functions or a manufacturing test menu
- Optional I2C and SPI debug logging enabled in firmware

### Safety notes

- Treat all heater and pump relay outputs as hazardous once mains is connected.
- Perform low-voltage validation first with mains disconnected.
- If validating mains switching, use an RCD-protected supply, fused test setup, and one hand where practical.
- Never probe live mains with an unearthed oscilloscope unless using a differential probe or isolated measurement method.
- Spring-return valves may move immediately when driven. Keep fingers clear.

---

## 2. Visual inspection checklist

Before applying power, inspect the PCB under good lighting and magnification.

### PCB and assembly

- Confirm the PCB revision matches the firmware and schematic being tested.
- Check for solder bridges, especially on the ESP32-S3 module pins, fine-pitch ICs, and connector footprints.
- Check for dry joints, lifted pads, tombstoned passives, and skewed IC placement.
- Confirm all polarized parts are fitted correctly:
  - Electrolytic capacitors
  - Diodes and TVS devices
  - Voltage regulators
  - Relays
  - Any transistor or MOSFET packages
- Confirm the ESP32-S3-WROOM-1U-N8R8 module is seated correctly with antenna clearance respected.
- Confirm the ADS131M08, PCA9635, PCF8574 devices, OLED connector, and pressure sensor are the expected fitted variants.

### Connector and wiring checks

- Verify screw terminals and headers are fully soldered and mechanically secure.
- Verify mains and SELV spacing is clean, with no solder splashes or debris.
- Confirm heater outputs are clearly labelled L1, L2, and L3.
- Confirm Jet Low, Jet High, Circulation/UV, Fill Valve, and Drain Valve outputs are labelled correctly.
- Confirm sensor and button connectors are populated in the correct orientation.

### Passive resistance checks before power

With power off:

- Measure resistance from 3.3 V to GND. A very low resistance, for example below roughly 10 ohms, usually indicates a short and must be investigated.
- Measure resistance from 5 V to GND if the rail exists locally.
- Measure resistance from 12 V to GND if the rail exists locally.
- Check there is no short between USB 5 V and GND.
- Check relay contact outputs are not shorted together unexpectedly.

A low charging resistance that rises as capacitors charge can be normal. A stable near-short is not.

---

## 3. Power-on checks

Apply power with current limiting if possible. Start with low-voltage power only and leave all mains loads disconnected.

### Initial power-up

- Set the bench supply current limit conservatively if powering externally.
- If powering from USB, watch for excessive inrush or a host overcurrent warning.
- Check for smoke, smell, overheating, or hot ICs within the first 10 to 20 seconds.

### Rail measurements

Measure all rails relative to board ground.

#### 3.3 V rail

- Expected: 3.3 V nominal
- Acceptable initial tolerance: 3.20 V to 3.40 V
- Preferred steady-state tolerance: within regulator specification, typically ±2% to ±3%
- Check ripple with oscilloscope if possible: ideally below 50 mVpp in idle bench conditions

#### 5 V rail

- Expected: 5.0 V nominal if generated or distributed on-board
- Acceptable range: 4.75 V to 5.25 V
- Check that USB-powered and externally powered conditions both remain within range if both modes are supported

#### 12 V rail

- Expected: 12 V nominal for valve or relay drive circuits if present
- Acceptable range for bench verification: 11.4 V to 12.6 V unless design specifies otherwise
- Verify the 12 V rail does not collapse when a valve or relay coil is energised during later steps

### Current consumption

If possible, record current draw in these states:

- Board powered, firmware idle
- OLED initialised
- LEDs enabled
- Relay coils enabled one at a time
- Wi-Fi active

A board drawing significantly more current than expected, especially with no outputs active, usually indicates a short, wrong part value, or firmware pin misconfiguration.

### If power checks fail

- If a rail is missing: inspect the upstream regulator, enable pin, fuse, reverse-polarity protection, and input connector.
- If a rail is low: look for shorts, excessive load, incorrectly fitted capacitors, or a regulator in thermal shutdown.
- If a rail is high: stop immediately and verify the regulator part number, feedback network, and assembly.
- If current is excessive: isolate sections by removing power to loads or depopulating suspect parts where possible.

---

## 4. MCU programming and flash verification

Verify that the ESP32-S3 can boot, flash, and run code reliably.

### USB and serial enumeration

- Connect the board to the host PC using a known-good data cable.
- Confirm the serial device enumerates.
- Open a serial terminal at the firmware’s configured baud rate, commonly 115200 baud for boot logs.

### Bootloader verification

Reset or power-cycle the board and capture the boot log.

Expected observations:

- ROM boot output is visible.
- Boot mode is normal, not stuck in download mode unless intentionally forced.
- No repeated brownout resets.
- Application boots and prints firmware version or hardware test banner.

### Flash test

- Put the board into download mode if required.
- Erase and flash the known hardware-test firmware image.
- Read back the flash status if needed.
- Reset and confirm the application starts without checksum or partition errors.

Example checks:

- `esptool.py chip_id` succeeds
- `esptool.py flash_id` returns a valid device
- `esptool.py erase_flash` completes normally
- Firmware flashing completes with no verification errors

### Functional verification

The test firmware should ideally provide:

- Serial command shell or test menu
- I2C bus scan command
- SPI register read for ADS131M08
- 1-Wire enumeration
- OLED text test
- Output toggle commands
- Input state readback
- Wi-Fi scan or ping test

### If MCU programming fails

- If no serial port appears: check USB cable, USB-UART path, ESD protection, and 5 V to 3.3 V power path.
- If flashing times out: check boot strap circuitry, EN pin behaviour, DTR/RTS auto-program circuit, and UART routing.
- If the board resets repeatedly: check 3.3 V rail stability, decoupling, flash configuration, and watchdog reset cause.
- If RF or PSRAM-related errors appear: confirm the correct ESP32-S3 module variant and matching firmware configuration.

---

## 5. I2C bus scan and device presence check

Verify the I2C bus before testing dependent peripherals.

### Preparation

- Ensure the firmware can perform an I2C scan and print detected addresses.
- Attach a logic analyser or oscilloscope if troubleshooting is required.
- Measure idle SDA and SCL levels. Both should sit high, typically near 3.3 V.

### Bus electrical checks

- SDA idle high: approximately 3.3 V
- SCL idle high: approximately 3.3 V
- Typical pull-up resistor values are often 2.2 kΩ to 10 kΩ depending on bus capacitance; check against schematic if uncertain.
- During activity, confirm clean edges and no lines stuck low.

### Expected I2C devices

At minimum, expect to detect:

- PCF8574 #0 at 0x40
- PCF8574 #1 at 0x42
- PCF8574 #2 at 0x44
- PCA9635 at approximately 0x65
- OLED display at its configured address
- SM3041-015-D-C-3-S pressure sensor at its configured address

If the OLED or pressure sensor address differs from the firmware expectation, record the actual address and update the configuration.

### Procedure

1. Boot the test firmware.
2. Run the I2C scan command.
3. Record all detected addresses.
4. If a device is missing, probe SDA and SCL while rescanning.
5. Confirm no address aliasing or bus lock-up occurs.

### Pass criteria

- All expected devices acknowledge.
- No unexpected bus contention.
- No device causes SDA to remain low after transactions.

### If I2C scan fails

- If no devices respond: check bus pull-ups, 3.3 V supply to peripherals, and that the MCU is using the correct pins.
- If only some devices respond: inspect the missing device’s soldering, address strap pins, reset pin, and local decoupling.
- If SDA or SCL is stuck low: isolate sections, look for shorts, and inspect the last device on the bus.
- If the scan is unstable: reduce bus speed, inspect waveform integrity, and check cable length or stub routing.

---

## 6. SPI bus verification for ADS131M08

Verify that the SPI bus and the ADS131M08 energy metering ADC communicate correctly.

### Signals of interest

- SCLK
- MOSI
- MISO
- CS for ADS131M08
- DRDY if connected
- RESET if exposed or firmware-controlled

### Electrical checks

- Confirm the ADS131M08 supply rails are present and within tolerance.
- Confirm CS idles inactive.
- Confirm SCLK is idle when no transfer is active.
- Use an oscilloscope or logic analyser to confirm clean SPI edges.

### Procedure

1. Boot the firmware with ADS131M08 support enabled.
2. Issue a device reset through firmware if available.
3. Read one or more known registers, such as ID, STATUS, or configuration registers.
4. Confirm the read values are consistent across repeated attempts.
5. If DRDY is used, confirm it toggles at the expected data rate after configuration.

### Expected behaviour

- The ADC should respond consistently to register reads.
- MISO should not remain tri-stated or fixed at 0xFF or 0x00 unless there is a bus fault.
- If DRDY is enabled, the period should match the configured sample rate within a few percent.

### Practical checks with a logic analyser

- Verify CS asserts before clocking starts.
- Verify the SPI mode matches the ADC requirement from the datasheet.
- Verify the command and response framing are correct.
- Verify no extra clocks are present at transaction boundaries.

### If SPI verification fails

- If all reads return 0xFF: check MISO continuity, CS polarity, and whether the ADC is powered.
- If all reads return 0x00: check reset state, clocking, and whether CS is permanently active.
- If reads are inconsistent: lower SPI speed, inspect ringing, and verify ground reference.
- If DRDY never toggles: check ADC reset, clock configuration, and firmware initialisation order.

---

## 7. 1-Wire bus and DS18B20 read test

Verify the DS18B20 temperature sensor on GPIO38.

### Electrical checks

- Measure GPIO38 idle level. It should sit high through the 1-Wire pull-up, typically near 3.3 V.
- Confirm the pull-up resistor is fitted. A typical value is 4.7 kΩ unless the design specifies another value.

### Procedure

1. Boot the hardware-test firmware.
2. Run a 1-Wire bus scan.
3. Confirm exactly one DS18B20 device is detected unless multiple devices are intentionally present.
4. Read the ROM code and record it.
5. Trigger a temperature conversion and read the temperature result.
6. Warm the sensor gently by hand or cool it slightly with ambient airflow and confirm the reading changes plausibly.

### Expected values

- At room temperature, a typical reading is around 15 °C to 30 °C depending on environment.
- The DS18B20 default power-up value of 85.0 °C indicates conversion has not completed or data is invalid.
- A reading of -127 °C or similar sentinel value usually indicates communication failure.

### If DS18B20 test fails

- If no device is found: check GPIO38 continuity, pull-up resistor, sensor power, and pinout.
- If the temperature is fixed at 85 °C: ensure conversion delay is long enough and scratchpad reads are valid.
- If the reading is erratic: inspect bus waveform, cable length, grounding, and pull-up strength.

---

## 8. OLED display initialisation test

Verify that the HS96L03W2C03 OLED display powers up and receives I2C commands.

### Procedure

1. Confirm the OLED I2C address appears in the bus scan.
2. Run a display initialisation test from firmware.
3. Display a known sequence:
   - Full-screen on or all pixels set
   - Full-screen off
   - Test pattern such as checkerboard or border
   - Text showing firmware version or test status
4. Observe power rail stability during updates.

### Pass criteria

- Display initialises without bus errors.
- Contrast is stable with no flicker beyond normal multiplex behaviour.
- The correct pattern and text appear.
- No missing rows, missing columns, or corrupted sections are visible.

### If OLED test fails

- If the device is present on I2C but the screen is blank: check reset sequencing, initialisation sequence, and OLED supply voltage.
- If the screen shows noise or partial image: check SDA/SCL integrity, connector orientation, and display RAM addressing configuration.
- If the display resets the board or drags down the rail: inspect power decoupling and inrush handling.

---

## 9. PCF8574 expander input and output test

Test all three PCF8574 expanders at 0x40, 0x42, and 0x44.

### Notes

The PCF8574 uses quasi-bidirectional I/O. Firmware should explicitly manage each pin according to whether it is used as an input or output. Review the schematic to map each expander pin to buttons, LEDs, relays, or interrupt-generating inputs.

### Procedure

1. Confirm all three addresses appear in the I2C scan.
2. For each expander, read back the current port value.
3. Write a walking pattern to output-configured pins, one bit at a time.
4. Verify the corresponding physical outputs change state.
5. For input-configured pins, actuate the connected switch or signal and verify the input bit changes.
6. Verify interrupt operation on:
   - GPIO8 for PCF8574 #0
   - GPIO9 for PCF8574 #2
7. Confirm the interrupt line idles high and pulses or latches low according to the expander behaviour when an input changes.

### Suggested output test pattern

For each expander output group:

- `0xFE`, `0xFD`, `0xFB`, `0xF7`, `0xEF`, `0xDF`, `0xBF`, `0x7F`
- Then `0xFF`

This pattern helps verify each bit independently.

### Pass criteria

- Every addressed expander responds reliably.
- Each controlled output maps to the correct physical function.
- Each input changes state correctly.
- Interrupt lines assert when expected and clear after the input status is read and the condition is removed.

### If PCF8574 testing fails

- If one expander is missing: check address strapping and soldering.
- If output states appear inverted: verify the external circuit polarity and firmware assumptions.
- If an interrupt line is stuck low: inspect which input changed last, confirm pull-up integrity, and read the port to clear the condition.
- If multiple functions move together: check for solder bridges or wrong net assignments.

---

## 10. PCA9635 lighting driver test

Verify the PCA9635 LED driver can control illuminated button LEDs or any other LED channels.

### Procedure

1. Confirm the PCA9635 address appears in the I2C scan.
2. Read back MODE1, MODE2, or any known register if firmware supports it.
3. Command all LED outputs off.
4. Command each LED output on at full duty cycle.
5. Run a PWM ramp, for example 0%, 25%, 50%, 75%, and 100%.
6. Observe each illuminated button LED: Drain, Fill, Light, and Jet.

### Expected behaviour

- LEDs switch cleanly on and off.
- PWM dimming is smooth with no obvious channel dropout.
- Brightness should roughly follow commanded duty cycle.
- No channel should ghost significantly when commanded fully off.

### Practical measurements

- Check LED supply voltage under load.
- Probe one LED output with an oscilloscope during PWM and confirm duty cycle changes as commanded.
- PWM frequency should remain stable and free from obvious jitter.

### If PCA9635 test fails

- If the device acknowledges but LEDs do not light: check LED polarity, current-limiting components, output enable logic, and LED supply.
- If only some channels work: inspect the failed output pins and associated LEDs.
- If brightness is wrong: verify resistor values and PCA9635 mode configuration.

---

## 11. Relay and actuator output verification

This section verifies heater relays, jet outputs, circulation/UV, and valves. Perform low-voltage coil-side checks first. Connect mains only after confirming control logic is correct.

### Safety warning

Heater outputs switch 230 VAC at up to 16 A per phase. This is hazardous energy. If you are not equipped for safe mains testing, stop after verifying the low-voltage control side.

### Outputs to verify

- Heater L1 relay
- Heater L2 relay
- Heater L3 relay
- Jet Low
- Jet High
- Circulation/UV
- Fill Valve, 12 V spring-return
- Drain Valve, 12 V spring-return

### Low-voltage control-side procedure

1. Keep mains disconnected.
2. Use firmware commands to toggle one output at a time.
3. Confirm relay coil voltage appears only on the selected channel.
4. Listen for a clean relay click where applicable.
5. Measure voltage at the fill and drain valve outputs when commanded on.
6. Confirm non-selected outputs remain inactive.

### Expected values

- Relay coil drive voltage should match the design rail, commonly 5 V or 12 V depending on the relay type.
- Valve drive output should match the nominal 12 V rail when active, within roughly ±5% under bench load.
- Coil supply should not dip enough to reset the MCU.

### Contact-side dry verification

With power off and no mains connected:

- Measure continuity across NO and COM contacts while toggling relay state.
- Verify the contact transitions from open to closed only when commanded.
- Confirm contact mapping matches the silk and schematic.

### Live mains verification, only if safe setup is available

- Use a fused, current-limited test load such as an incandescent lamp or suitable resistive load, not the final heater initially.
- Verify each heater phase relay switches its load independently.
- Verify Jet Low and Jet High are never energised together unless the design explicitly permits it.
- Verify Circulation/UV output switches correctly.

### Valve verification

- Command Fill Valve on and confirm 12 V is present and the actuator moves as expected.
- Remove command and confirm spring return.
- Repeat for Drain Valve.
- Check for excessive coil current or supply droop.

### If relay or actuator testing fails

- If a relay does not click: check driver transistor or MOSFET, flyback diode orientation, and coil supply.
- If a relay clicks but the contact does not switch: inspect the relay contact pins and part number.
- If the MCU resets during switching: inspect flyback suppression, rail decoupling, grounding, and supply current limit.
- If Jet Low and Jet High overlap incorrectly: treat as a critical fault and fix firmware or hardware interlock before further testing.

---

## 12. Input circuit verification

Verify all user inputs and safety inputs, including interrupts and direct GPIO inputs.

### Inputs to verify

- 4 illuminated buttons: Drain, Fill, Light, Jet
- 5 onboard navigation buttons: Up, Down, Left, Right, Enter
- Level sensor
- Lid sensor 1
- Lid sensor 2
- Heater fault input on GPIO48
- Interrupt lines from PCF8574 #0 on GPIO8 and PCF8574 #2 on GPIO9

### Procedure

1. Display or log the raw input state continuously in firmware.
2. Press each button individually and confirm the correct logical input changes.
3. Release the button and confirm it returns cleanly to idle.
4. Toggle each safety input using the real sensor or a safe bench stimulus.
5. For heater fault on GPIO48, inject the safe logic level specified by the design and confirm firmware reports the fault.
6. For PCF8574-backed inputs, confirm the corresponding interrupt line asserts.

### What to look for

- Correct mapping between physical input and reported software name
- No stuck inputs
- No cross-coupling where one button changes multiple bits
- No excessive bounce beyond what firmware debounce can handle

### Electrical expectations

- Idle input levels should match the design pull-up or pull-down network, typically within 0.2 V of the relevant rail.
- Active low inputs should fall below approximately 0.8 V when asserted.
- Active high inputs should rise above approximately 2.4 V on a 3.3 V logic domain, preferably close to 3.3 V.

### If input verification fails

- If an input is inverted: verify whether the circuit is active low and update firmware accordingly.
- If an input floats: inspect pull-up or pull-down resistors and connector continuity.
- If a safety input is noisy: inspect filtering, shielding, and debounce strategy.
- If GPIO48 does not detect heater fault: check protection components and voltage translation into the ESP32 input range.

---

## 13. Pressure sensor test

Verify the SM3041-015-D-C-3-S pressure sensor on I2C.

### Preparation

- Confirm the sensor appears in the I2C scan.
- Confirm the firmware can read raw pressure data and, ideally, converted engineering units.
- Check the sensor supply voltage against the datasheet requirement.

### Procedure

1. Record the pressure reading at ambient, no applied water pressure.
2. Apply a known small pressure stimulus if a safe bench method exists.
3. Confirm the reading changes in the correct direction and returns when the pressure is removed.
4. Repeat several times to check repeatability.

### Expected behaviour

- The sensor should provide stable readings at rest, with only small noise or quantisation changes.
- Raw readings should not saturate at minimum or maximum unless the sensor is misconfigured or faulty.
- Repeated readings under unchanged pressure should remain within a small band appropriate to the sensor resolution and firmware filtering.

### Practical acceptance guidance

- Ambient no-pressure reading should be consistent across repeated power cycles.
- A modest applied pressure should produce a clearly measurable delta, not random noise.
- If the firmware converts to physical units, confirm the value is plausible for the test condition.

### If pressure sensor test fails

- If the device is missing on I2C: inspect address, soldering, and supply.
- If raw data is constant or saturated: verify command format, byte order, and device mode.
- If readings drift excessively: inspect supply stability, mechanical connection, and firmware averaging.

---

## 14. Energy metering ADC test

After basic SPI communication is verified, validate analogue measurement behaviour of the ADS131M08.

### Preparation

- Ensure analogue front-end inputs are connected as intended.
- Confirm the ADC reference and analogue supplies are present and quiet.
- If using current transformers, shunts, or voltage dividers, verify they are fitted with the correct values.

### Procedure

1. With no intentional input signal, read all ADC channels and record baseline values.
2. Confirm offsets are reasonable and stable.
3. Apply a known low-level test signal to one channel at a time if safe and practical.
4. Confirm the selected channel responds with the expected polarity and relative magnitude.
5. Repeat for all populated channels.
6. If firmware computes RMS, voltage, current, or power, compare against a known reference source or bench instrument.

### Expected behaviour

- Unstimulated channels should be near zero or the expected offset baseline.
- Noise should be bounded and repeatable, not rail-to-rail.
- Applying a signal to one channel should not cause large response on unrelated channels.
- Computed energy values should be within the expected tolerance of the analogue front-end design and calibration state.

### Practical tolerances

Until calibration is complete, use loose engineering acceptance criteria:

- Channel offset stable over 10 seconds with no sudden jumps
- Relative amplitude tracking within a few percent between repeated measurements
- Cross-talk low enough that inactive channels remain clearly distinguishable from the active one

### If energy metering ADC test fails

- If one channel is saturated: inspect the front-end divider, gain path, and solder bridges.
- If all channels are noisy: inspect reference decoupling, ground layout, SPI integrity, and sample clock configuration.
- If channel order is wrong: verify firmware channel mapping.
- If computed power is negative or implausible: check phase polarity, sign convention, and calibration constants.

---

## 15. Wi-Fi basic connectivity test

Verify basic ESP32-S3 wireless functionality after the board passes wired subsystem checks.

### Procedure

1. Boot firmware with Wi-Fi enabled.
2. Scan for nearby access points.
3. Confirm expected SSIDs are listed with plausible RSSI values.
4. Connect to a known-good 2.4 GHz network supported by the ESP32-S3 configuration.
5. Confirm DHCP address assignment.
6. Ping a gateway or known host if the firmware supports it.
7. Optionally perform a simple HTTP request or MQTT test if available.

### Pass criteria

- Wi-Fi scan completes reliably.
- Association succeeds with correct credentials.
- The board obtains an IP address.
- Connectivity remains stable for at least 5 minutes with no spontaneous resets.

### If Wi-Fi test fails

- If no networks are seen: check antenna clearance, module variant, RF layout, and firmware region settings.
- If scan works but connection fails: recheck credentials, security mode, and access point compatibility.
- If the board resets under Wi-Fi load: inspect the 3.3 V rail for droop and review power supply headroom.

---

## 16. Error checks and troubleshooting summary

Use this quick triage guide whenever a step fails.

### General fault-isolation approach

1. Confirm the relevant supply rail first.
2. Confirm the MCU pin assignment and firmware build match the hardware revision.
3. Check whether the fault is local to one device or shared across a bus.
4. Use the simplest possible test stimulus.
5. Measure before replacing parts.

### Common failure patterns

#### Board does not boot

- Check 3.3 V rail
- Check EN and boot strap pins
- Check USB data path and serial port
- Check for shorts or overheating

#### I2C bus stuck or missing devices

- Check SDA and SCL idle levels
- Check pull-ups and device power
- Isolate the missing device
- Lower bus speed and retry

#### SPI device not responding

- Check CS polarity and routing
- Confirm supply and reset state of ADS131M08
- Inspect clock waveform and SPI mode

#### Inputs behave incorrectly

- Confirm active-high versus active-low assumptions
- Check pull resistors and connector orientation
- Review debounce and interrupt handling

#### Outputs switch incorrectly

- Verify output mapping in firmware
- Check transistor drivers, flyback diodes, and relay part numbers
- Confirm load supply remains in tolerance when switching

#### Sensor readings are implausible

- Confirm the device is detected on the expected bus
- Check conversion timing and register format
- Compare raw values before trusting engineering unit conversions

### Final acceptance checklist

A board can be considered ready for system integration when all of the following are true:

- Power rails are within tolerance and stable.
- ESP32-S3 flashes and boots reliably.
- All expected I2C devices are detected.
- ADS131M08 responds over SPI and produces plausible data.
- DS18B20 reads valid temperature values.
- OLED initialises and displays the test pattern.
- All PCF8574 expanders pass input and output checks.
- PCA9635 controls all intended LEDs correctly.
- Relay and valve outputs switch correctly without disturbing logic power.
- Buttons and safety inputs report correctly.
- Pressure sensor readings are plausible and repeatable.
- Wi-Fi scan and connection succeed.

### Recommended test record

For each tested board, record:

- PCB revision
- Serial number
- Firmware version and git commit if available
- Measured 3.3 V, 5 V, and 12 V rails
- Detected I2C addresses
- ADS131M08 register readback result
- DS18B20 ROM code and temperature reading
- OLED pass or fail
- Input and output pass or fail summary
- Pressure sensor baseline reading
- Wi-Fi RSSI and connection result
- Tester name and date

This record will make later failure analysis much easier.
