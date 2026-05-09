# 03 - I/O Mapping

## Overview

This file collects the currently confirmed logical I/O ownership for the Sommerhusspa controller.

## Input mapping

### Direct ESP32 inputs
- **GPIO48** = Heater fault / safety-chain input
- **GPIO38** = DS18B20 temperature sensor data line
- **GPIO8** = Interrupt from PCF8574 #0
- **GPIO9** = Interrupt from PCF8574 #2

### PCF8574 #0 inputs
- P0 = Level sensor
- P1 = Lid sensor 1
- P2 = Lid sensor 2
- P3 = Up
- P4 = Down
- P5 = Left
- P6 = Right
- P7 = Enter

### PCF8574 #2 button inputs
- P0 = Drain button
- P2 = Fill button
- P4 = Light button
- P6 = Jet button

### Other peripheral inputs
- Pressure sensor via I2C
- ADS131M08 phase voltage/current channels via SPI

## Output mapping

### PCF8574 #1 outputs
- P0 = Heater L1
- P1 = Heater L2
- P2 = Heater L3
- P3 = Jet High
- P4 = Jet Low
- P5 = Circulation / UV
- P6 = Valve Fill
- P7 = Valve Drain

### PCF8574 #2 LED outputs
- P1 = Drain button LED
- P3 = Fill button LED
- P5 = Light button LED
- P7 = Jet button LED

### PCA9635 lighting outputs
- LED0 = On/off lighting control output
- LED1 = Red PWM
- LED2 = Green PWM
- LED3 = Blue PWM

## Logical grouping

### Safety-related inputs
- Level sensor
- Heater fault input
- Lid sensors

### UI inputs
- Drain button
- Fill button
- Light button
- Jet button
- Up / Down / Left / Right / Enter

### Actuator outputs
- Heater L1/L2/L3
- Jet Low / Jet High
- Circulation / UV
- Fill valve
- Drain valve
- Button LEDs
- Lighting control outputs

## Notes

- PCF8574 #2 mixes input and output functions and should be treated carefully in firmware.
- The heater fault input is direct to the ESP32 rather than routed through an expander.
- Level sensor and heater fault are both operationally important for blocking hazardous operation.
