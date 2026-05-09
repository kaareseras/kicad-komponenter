# 01 - Hardware Overview

## Purpose

This document provides a high-level hardware overview of the Sommerhusspa controller platform. It is intended as a foundation for later firmware and system documentation.

The system is built around an ESP32-S3-based controller that manages heating, pumps, valves, user inputs, lighting, display/UI, temperature and pressure sensing, and three-phase energy metering.

## Main controller

- **MCU module:** ESP32-S3-WROOM-1U-N8R8
- **Primary roles:**
  - local standalone control
  - sensor acquisition
  - relay/output control
  - user interface handling
  - remote connectivity
  - diagnostics and metering integration

## Controlled loads and outputs

### High-power / mains-side loads
- **Heater**
  - 3 x 230 VAC / 16 A
  - one relay per phase: L1, L2, L3
- **Jet pump**
  - two command outputs:
    - Jet Low
    - Jet High
- **Circulation pump + UV**
  - intentionally combined on one output

### Low-voltage actuators
- **Fill valve**
  - 12 V motor valve
  - spring return to closed when de-energized
- **Drain valve**
  - 12 V motor valve
  - spring return to closed when de-energized

### Lighting-related outputs
Lighting is split into two functional domains:

1. **Spa lighting control via PCA9635**
   - one on/off lighting control output, used for external light-controller/test purposes
   - three RGB PWM outputs for direct RGB control

2. **Illuminated button indicators via PCF8574 #2**
   - Drain button LED
   - Fill button LED
   - Light button LED
   - Jet button LED

## Inputs and sensors

### User inputs
- **4 external illuminated buttons**
  - Drain
  - Fill
  - Light
  - Jet
- **5 onboard navigation buttons**
  - Up
  - Down
  - Left
  - Right
  - Enter

### Safety and state inputs
- Level sensor
- Lid sensor 1
- Lid sensor 2
- Heater fault / safety-chain input

### Process sensors
- **Water pressure sensor**
  - I2C
  - device: **SM3041-015-D-C-3-S**
- **Water temperature sensor**
  - 1-Wire
  - device: **DS18B20**

## Display and UI
- **OLED display**
  - I2C
  - device: **HS96L03W2C03**

## Metering and diagnostics
- **Energy metering ADC:** ADS131M08IPBSR
- **Bus:** SPI
- **Measurement topology:**
  - three phase-current channels
  - three phase-voltage channels
  - two unused channels

The metering subsystem is intended for:
- future energy management
- verification that commanded components are actually operating
- diagnostic and remote reporting functions

## Bus architecture summary

### I2C
- OLED display: HS96L03W2C03
- Pressure sensor: SM3041-015-D-C-3-S
- 3 x PCF8574 I/O expanders
- PCA9635PW,118 LED/light driver

### SPI
- ADS131M08IPBSR energy metering ADC

### 1-Wire
- DS18B20 water temperature sensor

## Connectivity features
- **Wi-Fi:** yes, for remote control and remote management
- **BLE:** yes, intended for commissioning only
- **USB:** present, but not used in normal operation
- **OTA updates:** yes

## Design intent

The controller should function as a locally autonomous spa controller while also supporting remote management and future higher-level energy features. The hardware architecture therefore combines:

- direct control of critical outputs
- local user interface and safety inputs
- remote connectivity
- metering-informed diagnostics
- future energy-aware control extensions
