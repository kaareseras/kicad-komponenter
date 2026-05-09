# 02 - MCU and Peripheral Mapping

## MCU

- **Main controller:** ESP32-S3-WROOM-1U-N8R8

The ESP32-S3 is the system master and owns the communication buses, control logic, UI logic, metering integration, and remote connectivity.

## Direct ESP32 GPIO assignments

Confirmed direct assignments:

- **GPIO8** = interrupt from PCF8574 #0
- **GPIO9** = interrupt from PCF8574 #2
- **GPIO38** = DS18B20 1-Wire water temperature sensor
- **GPIO48** = Heater fault input

Additional bus pins for I2C and SPI are expected but are not yet explicitly documented in this file.

## I2C devices

### OLED display
- **Device:** HS96L03W2C03
- **Role:** local display/UI output

### Pressure sensor
- **Device:** SM3041-015-D-C-3-S
- **Role:** water-pressure measurement used to estimate water level and stop filling

### LED / lighting driver
- **Device:** PCA9635PW,118
- **Role:** spa-lighting-related control outputs

Documented channel usage:
- **LED0** = on/off lighting control output for external light-controller/test use
- **LED1** = RGB Red PWM
- **LED2** = RGB Green PWM
- **LED3** = RGB Blue PWM

### I/O expanders
- **Devices:** 3 x PCF8574MS/TR

#### PCF8574 #0
- **Address:** write `0x40`, read `0x41`
- **Interrupt line:** GPIO8
- **Role:** local/safety/navigation inputs

Bit mapping:
- P0 = Level sensor
- P1 = Lid sensor 1
- P2 = Lid sensor 2
- P3 = Up
- P4 = Down
- P5 = Left
- P6 = Right
- P7 = Enter

#### PCF8574 #1
- **Address:** write `0x42`, read `0x43`
- **Role:** relay and actuator control

Bit mapping:
- P0 = Heater L1
- P1 = Heater L2
- P2 = Heater L3
- P3 = Jet High
- P4 = Jet Low
- P5 = Circulation / UV
- P6 = Valve Fill
- P7 = Valve Drain

#### PCF8574 #2
- **Address:** write `0x44`, read `0x45`
- **Interrupt line:** GPIO9
- **Role:** external buttons and button LEDs

Bit mapping:
- P0 = Drain button
- P1 = Drain button LED
- P2 = Fill button
- P3 = Fill button LED
- P4 = Light button
- P5 = Light button LED
- P6 = Jet button
- P7 = Jet button LED

## Important firmware note about PCF8574 #2

PCF8574 #2 is a **mixed-direction expander**:
- some bits are used as button inputs
- some bits are used as LED outputs

Because PCF8574 uses quasi-bidirectional I/O, firmware should maintain a shadow register/state model so that output bits are preserved while reading input states.

## SPI devices

### ADS131M08IPBSR
- **Role:** three-phase energy metering ADC

Confirmed channel map:
- CH0 = Current L3
- CH1 = Voltage L3
- CH2 = Current L2
- CH3 = Voltage L2
- CH4 = Current L1
- CH5 = Voltage L1
- CH6 = Not connected
- CH7 = Not connected

This arrangement provides matched current/voltage pairs per phase.

## 1-Wire devices

### DS18B20
- **GPIO:** GPIO38
- **Role:** water temperature measurement before the heater

## Functional ownership summary

### Owned through expanders
- heater relays
- jet low/high outputs
- circulation/UV output
- fill/drain valve outputs
- external illuminated-button inputs
- illuminated-button LED indicators
- level and lid-related inputs
- onboard UI buttons

### Owned through dedicated peripherals
- spa lighting control: PCA9635
- pressure measurement: SM3041 over I2C
- temperature measurement: DS18B20 over 1-Wire
- display: OLED over I2C
- energy metering: ADS131M08 over SPI

### Owned directly by ESP32 GPIO
- heater fault input
- PCF interrupt inputs
- 1-Wire bus for water temperature
