# 11 - Bill of Materials and Sourcing

## 1. Overview and format

This document is the working Bill of Materials (BOM) and sourcing guide for the Sommerhusspa controller.

It is written to support three parallel goals:

- schematic capture and PCB layout
- practical purchasing from LCSC, DigiKey, Mouser, and similar distributors
- manufacturing planning, especially JLCPCB SMT assembly versus hand assembly

The project is a mixed-voltage design intended for a 230 V spa environment. That means the BOM must be treated in three classes:

1. **logic and measurement electronics** on the low-voltage PCB
2. **field wiring and operator interface hardware** that may be panel-mounted or cabled
3. **mains switching and safety components** that must be selected for 16 A / 250 VAC minimum where they switch heater or other mains loads

Where a supplier code was directly identifiable from known distribution listings, it is included. Where an exact LCSC code could not be verified from the current project context, an alternative distributor part number or a note is provided instead. Any unverified supplier code should be confirmed again at ordering time.

---

## 2. Complete BOM table

| Ref. | Description | Manufacturer | Manufacturer part number | Supplier(s) / order codes | Qty. | Notes | JLCPCB assembly suitability |
|---|---|---|---|---|---:|---|---|
| U1 | Wi-Fi/BLE MCU module, ESP32-S3, 8 MB flash, 8 MB PSRAM, U.FL antenna | Espressif | ESP32-S3-WROOM-1U-N8R8 | LCSC: **C2980300**; DigiKey: **1965-ESP32-S3-WROOM-1U-N8R8-ND** or listing for ESP32-S3-WROOM-1U-N8R8 | 1 | Confirm external 2.4 GHz antenna and U.FL/I-PEX cable | Yes, if LCSC stock is available; module placement is JLC-friendly |
| U2, U3, U4 | 8-bit I2C I/O expander, TSSOP-16 | HGSEMI or NXP-compatible source | PCF8574MS/TR | LCSC: **C42382078** for HGSEMI PCF8574MS/TR; DigiKey reference family: NXP/TI PCF8574 equivalents | 3 | Low-cost expander used for inputs, outputs, and illuminated button I/O; address strapping required | Yes, JLC-friendly SMD |
| U5 | 16-channel I2C LED driver, TSSOP-28 | NXP | PCA9635PW,118 | LCSC: **C110795**; DigiKey: **568-4067-1-ND** | 1 | Drives RGB channels and auxiliary light-control outputs | Yes, JLC-friendly SMD |
| U6 | 8-channel simultaneous-sampling energy metering ADC, TQFP/QFN package per design footprint | Texas Instruments | ADS131M08IPBSR | DigiKey/Mouser: search exact MPN **ADS131M08IPBSR**; verify TI package and reel variant before ordering | 1 | Critical metering IC; pair with precision reference network, anti-alias filters, and isolated sensing front end as required | Usually not a standard JLC basic part; possible by SMT if stocked, otherwise hand-source and consign |
| U7 | 1-Wire temperature sensor, TO-92 or cable probe variant | Maxim / Analog Devices | DS18B20+ or DS18B20+T&R | LCSC: **C9753** for DS18B20+; LCSC: **C880672** for DS18B20+T&R listing; DigiKey/Mouser widely stocked | 1 | For spa water temperature; remote probe or stainless waterproof leaded assembly may be preferred over bare TO-92 | Bare TO-92 is not ideal for standard SMT assembly; panel/cable probe version is hand-installed |
| U8 | I2C pressure sensor, 15 psi differential/compound variant | Silicon Microstructures, Inc. (SMI) | SM3041-015-D-C-3-S | Chip1Stop or direct franchise distributor recommended; confirm availability by exact MPN | 1 | Core process sensor for fill-level inference; verify supply voltage, I2C address, and wetted-media compatibility | Likely hand-sourced; assembly depends on package and distributor availability |
| DS1 | I2C OLED display module | Hantronix or module vendor | HS96L03W2C03 | Supplier to be confirmed; likely module-level procurement rather than reel distribution | 1 | Usually purchased as a display module, not as a bare SMT glass component | No, normally hand-assembled or cabled |
| K1, K2, K3 | Power relay or contactor output, heater phase switching, minimum 16 A / 250 VAC | Finder or Hongfa | Example: Finder **40.52.9.230.0000** or Hongfa **HF115F/012-1ZS3** equivalent control-side selection must match PCB drive voltage | 3 | One per heater phase; use relay/contact arrangement appropriate to coil voltage and creepage requirements | Usually hand-assembled; through-hole or DIN-rail contactor approach recommended |
| K4 | Jet pump high-speed command relay | Finder / Hongfa / Omron | Example: Finder 40-series or Omron G2R / G5LE family, minimum load rating to match pump command circuit | 1 | Verify whether this is logic interlock control or direct mains switching | Usually hand-assembled |
| K5 | Jet pump low-speed command relay | Finder / Hongfa / Omron | Example: same family as K4 | 1 | For dual-speed pump; firmware and wiring must guarantee no simultaneous low/high energisation | Usually hand-assembled |
| K6 | Circulation / UV output relay | Finder / Hongfa / Omron | Example: same family as K4, sized to actual load | 1 | Combined output by project definition | Usually hand-assembled |
| Q1, Q2, Q3, Q4, Q5, Q6 | Relay / valve driver transistor or low-side MOSFET | Diodes Inc., Nexperia, onsemi, etc. | Example: **AO3400A** or **BC817-40** depending on load | LCSC widely available; select once coil/valve current is final | 6 | Needed for heater relays, jet outputs, circulation output, fill/drain valve drive depending on architecture | Yes, very JLC-friendly once selected |
| D1-D6 | Flyback diodes for relays/valves | Diodes Inc., MCC, onsemi, etc. | Example: **SS14**, **S1M**, or **1N4148WS** depending on load and topology | LCSC widely available | 6 | Use across inductive loads; exact part depends on coil current and supply rail | Yes |
| PS1 | AC/DC power supply module for logic rail | Mean Well / CUI / Hi-Link | Example: Mean Well **IRM-10-12** or Hi-Link **HLK-10M12** | Mouser, DigiKey, TME, LCSC for some variants | 1 | Recommended to generate isolated 12 V from mains, then derive 5 V and 3.3 V locally | Usually hand-assembled; observe isolation and approvals |
| U9 | 12 V to 5 V buck regulator | Monolithic Power / TI / Richtek / Silergy | Example: **MP1584EN** module or PCB IC such as **TPS54202DDCR** | LCSC/Mouser/DigiKey widely available | 1 | Required if 5 V rail is used for display, LEDs, or sensor margin | Yes if implemented with SMT IC, not if using external power module |
| U10 | 5 V to 3.3 V regulator, LDO or buck | Diodes Inc., TI, Microchip, Torex | Example: **AP2112K-3.3**, **TLV75533PDBVR**, or **MCP1700T-3302E/TT** | LCSC widely available | 1 | Powers ESP32 and 3.3 V logic; thermal budget must be checked | Yes |
| J1 | U.FL / I-PEX antenna connector or module mating cable | Hirose-compatible | U.FL-compatible | LCSC/Mouser/DigiKey | 1 | Needed because the selected ESP32 module is the 1U variant with external antenna | Usually hand-assembled or placed by SMT if footprint is on PCB |
| ANT1 | 2.4 GHz adhesive or whip antenna with U.FL cable | Various | Example: 2.4 GHz Wi-Fi/BLE antenna, 50 ohm, U.FL | Mouser, DigiKey, Taoglas, TE, etc. | 1 | Mount away from mains harness and metal enclosure walls | No, off-board part |
| SW1-SW4 | Panel button, illuminated, momentary | E-Switch / Bulgin / Apem / generic industrial | Example family to be selected by panel hole size | 4 | External user controls: Drain, Fill, Light, Jet; use IP65 or better on panel | No, panel hardware |
| LEDA1-LEDA4 | LED source for illuminated buttons, if not integrated | Kingbright / Everlight / Lite-On | Example 3 mm / 5 mm or panel integrated | Supplier depends on chosen button family | 4 | Omit if illuminated button assemblies include integrated LEDs | Usually hand-installed or not separate |
| SW5-SW9 | PCB navigation tact switches | C&K / ALPS / XKB / E-Switch | Example 6 x 6 mm tact switch, SMT or THT | LCSC widely available | 5 | Up, Down, Left, Right, Enter; choose sealed version if board is exposed to humidity | Yes if SMT |
| S1 | Level sensor input device | Project-specific | Example: float switch, conductive probe, or optical level switch | Source according to mechanical tank design | 1 | Must be fail-safe and suitable for wet spa environment | Usually off-board, hand-wired |
| S2, S3 | Lid / cover position sensor | Honeywell / C&K / generic reed or magnetic sensor | Example magnetic reed switch set, IP-rated | Mouser, DigiKey, industrial suppliers | 2 | Use closed-loop detection with tamper-resistant mounting | Off-board, hand-wired |
| S4 | Heater fault / safety-chain input terminal | Phoenix Contact / WAGO / Degson | Example pluggable terminal block | LCSC/Mouser/DigiKey | 1 | Interface only; actual thermal cutoff / pressure / flow safety chain is external and mandatory | Terminal can be JLC-friendly if SMT/THT stocked |
| Y1 | 40 MHz crystal or oscillator for ESP32, if not internal to module usage | N/A for module | Not required for module itself | N/A | 0 | ESP32 module already contains required RF support parts | N/A |
| RPU1 | 1-Wire pull-up resistor | Yageo / UniOhm | **4.7 kΩ 1% 0603** | LCSC widely available | 1 | Required for DS18B20 bus | Yes |
| R_I2C1, R_I2C2 | I2C pull-up resistors | Yageo / UniOhm | **4.7 kΩ 1% 0603** | LCSC widely available | 2 | Fit once per bus rail set; adjust if total pull-up strength is too low with display module included | Yes |
| TVS1 | TVS diode on 12 V or field I/O rail | Littelfuse / Bourns / Semtech / MCC | Example **SMBJ15A** or **SMBJ18A** depending on rail | LCSC/Mouser/DigiKey | 1-3 | Recommended for noisy actuator wiring and external harness protection | Yes if SMB/SMC package fits assembly plan |
| TVS2 | Ethernet-style or signal-line TVS for sensor/button lines | Nexperia / Littelfuse | Example **ESD9M5V** / **SMF05C** | LCSC/Mouser/DigiKey | 1-4 | Recommended on exposed UI and sensor lines | Yes |
| OC1-OC6 | Optocouplers or digital isolators for mains interface, if galvanic isolation is used for output drive or status sensing | Vishay / Everlight / Toshiba / TI / ADI | Example **EL357N**, **TLP291**, or digital isolator family | LCSC/Mouser/DigiKey | As needed | Strongly recommended where low-voltage PCB interacts with mains-derived signals | Yes for many SMT optocouplers |
| TB1-TB8 | Field wiring terminal blocks | Phoenix Contact / WAGO / Degson | Example 5.08 mm pluggable terminal family | LCSC/Mouser/DigiKey | As needed | Separate low-voltage and mains terminals physically on PCB/enclosure | Usually hand-assembled |
| F1-F4 | Fuses for mains branches and PSU input | Littelfuse / Eaton / Schurter | Value depends on branch current and standards approach | Mouser, DigiKey | As needed | Heater branches may instead use upstream breaker/contactors; coordinate protection scheme | Hand-assembled |
| MOV1 | Mains surge suppressor | Bourns / EPCOS / Littelfuse | Example **MOV-14D471K** or approved equivalent | LCSC/Mouser/DigiKey | 1 | Recommended on AC input | Hand-assembled |
| NT1 | Inrush limiter for AC input, if PSU design needs it | Ametherm / EPCOS | Example NTC limiter sized to PSU | Mouser, DigiKey | 0-1 | Needed only if using internal AC/DC PSU topology requiring inrush control | Hand-assembled |
| ENC1 | Enclosure | Fibox / Hammond / Spelsberg / ROLEC | Example IP65 polycarbonate enclosure | Mouser, RS, local industrial suppliers | 1 | Prefer non-metallic enclosure for easier RF and insulation management | No |
| ST1-ST4 | PCB standoffs / mounting hardware | Würth / generic | M3 nylon or metal standoffs | Any hardware supplier | 4 | Keep clearance from enclosure floor and support heavy field wiring | No |
| Cx/Rx/filter network | Metering front-end passives | Various | Per ADS131M08 analogue design | LCSC/Mouser/DigiKey | Many | Includes burden resistors, dividers, RC filters, reference decoupling, anti-alias network | Mostly yes |

---

## 3. Major component sections

### 3.1 Power supply section

The confirmed component list does not yet lock the power architecture, but a practical implementation for this controller is:

- **230 VAC in**
- **isolated 12 V AC/DC module** for relays, valves, and intermediate power
- **5 V regulator** for display or auxiliary devices if needed
- **3.3 V regulator** for ESP32, I2C logic, ADC digital rail, and low-voltage sensors

Recommended concrete options:

- **AC/DC module:** Mean Well IRM-10-12, IRM-15-12, or approved equivalent.
- **12 V to 5 V buck:** TPS54202DDCR, MP1584-based design, or similar.
- **5 V to 3.3 V LDO:** AP2112K-3.3 or TLV75533.

Important points:

- Use an **isolated mains PSU**, not a capacitive dropper.
- Budget extra current for relay coils, valve inrush, button LEDs, display, and Wi-Fi transmit peaks.
- Separate high-current actuator rail routing from the MCU analogue and RF area.
- Add reverse-polarity and surge protection on the 12 V distribution if there are external harnesses.

### 3.2 MCU section

**Required parts:**

- U1: ESP32-S3-WROOM-1U-N8R8
- antenna connector and antenna
- 3.3 V regulator
- EN/BOOT support parts
- USB/UART programming interface if desired for development
- local decoupling capacitors

Practical notes:

- Because this is the **1U** module, the design requires an **external 2.4 GHz antenna**.
- Keep the antenna away from relay coils, mains terminals, shields, and enclosure metalwork.
- Add test pads for UART, reset, boot, 3.3 V, and ground.

### 3.3 Communication and peripheral section

#### I2C expanders

- 3 x **PCF8574MS/TR**
- One expander is input-only in practice.
- One expander is output-only in practice.
- One expander is mixed input/output and needs a firmware shadow register strategy.

Recommended alternatives if unavailable:

- **NXP PCF8574T/3,512**
- **TI PCF8574DWR** or equivalent package variant
- **PCAL9535A** if a redesign to 16-bit expander is acceptable

#### LED driver

- **PCA9635PW,118**
- Good fit for button or lighting PWM because it is a dedicated I2C LED driver with multiple outputs.

Alternatives:

- **PCA9532D,118** if simple dimming/output control is sufficient
- **TLC59116** family from TI if software migration is acceptable

#### Energy metering ADC

- **ADS131M08IPBSR**
- This is a serious analogue part and should not be treated like a generic utility ADC.

Additional required support parts around the ADC typically include:

- precision resistor dividers for phase voltage channels
- burden resistors or conditioned outputs from current transformers / shunts
- RC anti-alias filters on each channel
- voltage reference decoupling and analogue ground discipline
- optional isolation, depending on sensing topology

Recommended alternatives if unavailable:

- **ADS131M06** if channel count can be reduced
- **ATM90E32AS** if the architecture moves toward a more integrated metering front end
- **ADE9000** family if supply chain or measurement strategy changes

### 3.4 Input section

#### Temperature sensing

- **DS18B20** is confirmed.
- For a spa, the most practical procurement choice is usually a **sealed stainless waterproof DS18B20 probe assembly** with PTFE or PVC cable, rather than a bare TO-92 device.

#### Pressure sensing

- **SM3041-015-D-C-3-S** is confirmed.
- Verify:
  - pressure range versus actual water column and plumbing behaviour
  - package sealing and tubing interface
  - long-term moisture compatibility
  - supply rail compatibility with the rest of the design

#### Buttons and discrete inputs

- 4 external illuminated momentary buttons
- 5 local navigation buttons
- level sensor
- 2 lid sensors
- heater fault / safety chain input

Recommendations:

- Use **IP65 or better** panel buttons.
- Use **sealed tact switches** if the PCB is in a humid compartment.
- Use pull-ups, RC filtering, and ESD protection on all off-board switch inputs.
- Treat the heater fault and water-level-related inputs as safety-relevant signals.

### 3.5 Output section

#### Heater switching

There are three heater outputs, one per phase. For a 230 V spa system, every mains-switched component here should be rated at least **16 A / 250 VAC**.

Recommended implementation approach:

- low-voltage PCB drives **relay coils or contactor coils**
- mains current is switched by **power relays/contactors** with sufficient creepage, isolation, and approvals
- include snubbers or MOV suppression as appropriate for the load and relay contacts

If the heater current is close to 16 A continuously, a **DIN-rail contactor solution** may be better than putting all heater switching on the PCB.

#### Jet pump outputs

- Two command outputs: **Jet Low** and **Jet High**.
- Use mechanical or firmware interlock so both cannot be active at the same time.
- Confirm whether the spa pump expects switched mains, contact closure, or low-voltage control input.

#### Circulation / UV output

- One combined output.
- Size relay contacts to the actual pump + UV ballast/load profile.

#### Fill and drain valves

- 12 V motor valves with spring return to close.
- Use low-side MOSFET or relay drive according to actual actuator current and stall current.
- Add flyback suppression and preferably a fuse per actuator branch.

### 3.6 Display section

- **HS96L03W2C03** I2C OLED display module.
- Treat this as a module-level sourced item.
- Confirm display supply voltage, pin pitch, visible area, and mounting style.

Good practice:

- connect over a locking cable or board-to-board connector if mounted off the main PCB
- avoid placing the OLED directly in a condensation path
- if front-panel mounted, use gasketed mechanical support or a sealed window

### 3.7 Mechanical and enclosure section

Recommended enclosure class:

- **IP65 or better** polycarbonate or ABS industrial enclosure
- internal separation between mains wiring and low-voltage electronics
- gland entries with strain relief for pumps, heater, valves, sensors, and UI harnesses

Practical enclosure candidates:

- Fibox polycarbonate enclosures
- Hammond 1554 / 1555 family
- Spelsberg industrial control boxes

Recommended mechanical BOM items:

- pluggable terminal blocks for field serviceability
- nylon or metal M3 standoffs
- cable glands sized for mains and sensor harnesses
- internal divider or barrier between mains and SELV sections

---

## 4. Sourcing notes

### 4.1 JLCPCB-assembly-friendly parts

The following are the most likely to be suitable for standard SMT assembly if stocked by LCSC/JLCPCB:

- ESP32-S3-WROOM-1U-N8R8
- PCF8574MS/TR
- PCA9635PW,118
- 3.3 V regulator
- buck regulator ICs in standard SMT packages
- resistor/capacitor networks
- MOSFETs/transistors for low-side drive
- flyback diodes
- ESD/TVS protection parts
- most ADC support passives

### 4.2 Parts likely to require hand assembly or off-board procurement

These parts are less likely to fit a straightforward JLC SMT workflow:

- HS96L03W2C03 OLED display module
- waterproof DS18B20 probe assembly
- SM3041 pressure sensor, depending on package and source
- mains relays/contactors
- large terminal blocks and fuses
- illuminated panel buttons
- lid sensors, level sensor, and harnessed field devices
- enclosure, glands, standoffs, and panel hardware
- antenna and U.FL cable

### 4.3 Recommended alternatives for out-of-stock parts

#### MCU module
- Primary: ESP32-S3-WROOM-1U-N8R8
- Alternative: ESP32-S3-WROOM-1U-N8 or N16R8 variants, but only if memory requirements and firmware partitioning are revalidated

#### I/O expanders
- Primary: PCF8574MS/TR
- Alternatives: NXP/TI PCF8574 package variants; PCAL9535A if redesign is acceptable

#### LED driver
- Primary: PCA9635PW,118
- Alternatives: TLC59116, PCA9532, PCA9633 depending on required channel count and PWM needs

#### Temperature sensor
- Primary: DS18B20
- Alternatives: DS18S20 for legacy compatibility, or a sealed NTC probe plus ADC input if redesign is acceptable

#### Pressure sensor
- Primary: SM3041-015-D-C-3-S
- Alternatives: Honeywell, TE Connectivity, or NXP pressure sensors only if interface, range, and media compatibility are requalified

#### Heater relays/contactors
- Primary: Finder / Hongfa / Omron industrial relay families
- Alternatives: DIN-rail modular contactors from Schneider, Finder, Eaton, or Lovato if PCB relay thermal/mechanical limits are not comfortable

### 4.4 Environmental and quality grades

Recommended grade policy:

- **Industrial temperature grade preferred** for PCB ICs in an equipment enclosure near heated water.
- **Consumer/commercial grade may be acceptable** for non-critical UI parts if the enclosure temperature remains controlled.
- **Use UL/TÜV/VDE approved mains components** wherever they touch line voltage or heater switching.
- **Use sealed or IP-rated field parts** for buttons, sensors, and cable entries.

Specifically:

- choose relays/contactors with clear AC load ratings, not only resistive DC marketing numbers
- choose terminal blocks with voltage and current approvals matching the installation
- choose sensors with humidity and condensation tolerance appropriate for spa plant rooms

---

## 5. Optional versus required components

### 5.1 Required for first safe power-up of the real controller

These are effectively mandatory:

- ESP32-S3 module and 3.3 V power chain
- pull-ups and basic decoupling
- at least one I/O expander if inputs/outputs are being exercised
- heater fault / safety-chain input path
- level sensor input path
- properly rated output drive components
- mains PSU with isolation
- fusing / surge suppression appropriate to the installation
- relay/contactors or safe dummy-load interface
- isolation/clearance barriers and field terminals

### 5.2 Optional for early bring-up or bench testing

These can be omitted in a reduced prototype:

- OLED display
- pressure sensor
- external illuminated buttons
- RGB lighting driver outputs
- full three-phase metering front end
- fill and drain valves, replaced by test lamps or dummy loads
- lid sensors, if firmware permits a test mode

### 5.3 Must be included for safety compliance direction

For any installation approaching real-world use, the following should not be omitted:

- external safety chain input for heater inhibit
- low-water or no-water protection path
- properly rated mains switching devices
- branch protection and insulation strategy
- separation between mains and SELV
- protective enclosure and cable strain relief
- suppression for inductive loads
- reliable interlock preventing jet low and jet high simultaneous activation

---

## 6. Mechanical considerations

### 6.1 Enclosure suggestions

Recommended enclosure style:

- wall-mounted **IP65 polycarbonate enclosure**
- minimum two internal zones:
  - mains switching and line terminals
  - low-voltage controller and UI/sensor terminations

If Wi-Fi performance matters, prefer:

- non-metallic enclosure, or
- an external antenna feed-through if a metal box must be used

### 6.2 PCB mounting holes and form factor

Recommended PCB mechanical baseline:

- 4 mounting holes, **M3**, one near each corner
- keep at least 5-8 mm copper keepout around mounting hardware, more near mains nets
- target board size roughly **120 mm x 160 mm to 160 mm x 220 mm**, depending on whether mains relays stay on-board
- if heater switching stays on-board, expect a larger PCB and wider creepage slots

Layout guidance:

- low-voltage logic on one side of the board
- mains terminals and power relays/contactors on the opposite side
- routed isolation slot or physical gap between the two zones
- keep the ESP32 antenna edge near an enclosure wall free of copper and cable bundles

### 6.3 Button and display cutout dimensions

Exact dimensions depend on the final selected panel hardware, but practical starting points are:

#### External illuminated buttons
- Common panel-button sizes: **16 mm, 19 mm, or 22 mm mounting hole**
- Recommendation: **19 mm IP65 momentary illuminated buttons** for gloved/wet-hand use
- Typical spacing: at least **25-30 mm centre-to-centre**

#### Navigation buttons
- If PCB-mounted behind an overlay, use 6 x 6 mm tact switches with cap plungers
- Allow at least **10-12 mm pitch** for comfortable navigation cluster spacing

#### OLED display
- Because HS96L03W2C03 is a module-level part, confirm the datasheet before freezing the front panel
- As a design placeholder, reserve:
  - module keepout around **30 mm x 30 mm to 40 mm x 40 mm**
  - visible window slightly smaller than the active area
  - 2-4 mounting points or an adhesive carrier, depending on module style

Recommendation:

- do not machine the production enclosure until the actual display module is in hand and measured

---

## 7. Estimated cost range

These are rough planning figures only and depend strongly on distributor, stock region, and whether mains switching is PCB relay based or contactor based.

### 7.1 Low-volume prototype build, approximately 1-5 units

- ESP32-S3 module: **USD 3.5-6**
- 3 x PCF8574: **USD 0.8-3 total**
- PCA9635: **USD 1-2**
- ADS131M08: **USD 6-12** typical order-of-magnitude
- DS18B20 sensor/probe: **USD 1-8** depending on bare IC versus sealed probe
- SM3041 pressure sensor: **USD 15-40** depending on distributor and package
- OLED module: **USD 4-12**
- power supply chain: **USD 6-20**
- relays/contactors and drivers: **USD 20-60**
- buttons, terminals, enclosure, glands, hardware: **USD 30-90**
- passives, protection, connectors, PCB assembly overhead: **USD 20-60**

**Prototype total estimate:** roughly **USD 110-300** per controller excluding wiring loom and certification effort.

### 7.2 Small-batch build, approximately 25-100 units

With volume purchasing and a cleaner DFM pass:

- logic PCB electronics can drop noticeably
- enclosure and industrial hardware still dominate cost
- mains switching hardware remains a major line item

**Small-batch total estimate:** roughly **USD 80-220** per unit, again excluding labour-intensive wiring and compliance costs.

---

## 8. Quick-order suggestions

The following links or search patterns are useful starting points for purchasing. Replace with the final approved vendor list once procurement is frozen.

### Core ICs

- ESP32-S3-WROOM-1U-N8R8, LCSC search / part page: `https://www.lcsc.com/product-detail/C2980300.html`
- PCA9635PW,118, LCSC: `https://www.lcsc.com/product-detail/C110795.html`
- PCF8574MS/TR, LCSC HGSEMI source: `https://www.lcsc.com/product-detail/C42382078.html`
- DS18B20+, LCSC: `https://www.lcsc.com/product-detail/C9753.html`

### Distributor search strings worth saving

- `ADS131M08IPBSR site:mouser.com`
- `ADS131M08IPBSR site:digikey.com`
- `SM3041-015-D-C-3-S distributor`
- `HS96L03W2C03 display module`
- `Finder 16A 250VAC relay 12VDC coil`
- `Hongfa 16A 250VAC relay 12VDC coil`
- `19mm illuminated IP65 momentary pushbutton 12V LED`
- `polycarbonate IP65 enclosure 160x200x90`

### Recommended procurement split

A sensible ordering split for this project is:

- **LCSC/JLCPCB:** MCU module, expanders, LED driver, regulators, passives, transistors, TVS, small connectors
- **Mouser/DigiKey/Chip1Stop:** ADS131M08, SM3041, branded relays/contactors, approved PSU, industrial terminals, enclosure hardware
- **Industrial/local suppliers:** enclosure, glands, panel buttons, DIN components, waterproof sensors

---

## Final purchasing notes

Before placing the first serious order, confirm the following items explicitly:

1. **coil voltage** for every relay or contactor
2. **actual jet pump command interface**: mains, dry contact, or low-voltage logic
3. **current draw and stall current** of the fill and drain valves
4. **pressure sensor package and tubing/mechanical interface**
5. **OLED module dimensions and supply voltage**
6. **whether the heater switching remains on-PCB or moves to external contactors**
7. **creepage/clearance strategy** for the full 230 V design

If those seven points are locked, this BOM can be turned from a sourcing guide into a production-purchase BOM with much tighter manufacturer and supplier line items.
