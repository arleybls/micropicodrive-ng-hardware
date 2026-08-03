# QL MicroPicoDrive SPI — Schematic Documentation

**KiCad Version:** 20211123 (KiCad 6/7 format, compatible with KiCad 10)
**Board paper size:** A4 (both schematics)

---

## Project Overview

The QL MicroPicoDrive is a Sinclair QL MicroDrive emulator built around a Raspberry Pi Pico (RP2040). It emulates a MicroDrive cartridge drive unit, storing data on an SD card. The system consists of two PCBs connected via a 16-pin edge connector:

| Board | File | Description |
|---|---|---|
| **MicroPicoDrive** | `MicroPicoDrive.kicad_sch` | Main controller board — RP2040 Pico, power regulation, level shifters, QL bus interface |
| **MicroPicoDriveCartridge** | `MicroPicoDriveCartridge.kicad_sch` | Cartridge board — SD card, OLED screen, navigation buttons, LED indicators |

---

## 1. MicroPicoDrive — Main Controller Board

### 1.1 Power Architecture

| Rail | Source | Usage |
|---|---|---|
| **+9V** | External input (via J2 QL connector) | Input to LM7805 regulator |
| **+5V** | LM7805 (U1) output | QL bus logic, level-shifter VCCB side |
| **+3.3V** | Pico internal 3.3V regulator (pin 36) | Pico logic, level-shifter VCCA side |
| **GND** | Common ground | — |

**U1 — LM7805_TO220** (`TO-220-3_Vertical`): 5V linear regulator, 1A max. Input from +9V rail, output +5V rail.

Decoupling capacitors **C1–C6**: 100 nF, SMD 0603 (`Capacitor_SMD:C_0603_1608Metric`), placed throughout the board for supply bypassing.

### 1.2 Component Bill of Materials

| Ref | Value / Part | Description | Footprint |
|---|---|---|---|
| **U1** | LM7805_TO220 | 5V 1A linear voltage regulator | `Package_TO_SOT_THT:TO-220-3_Vertical` |
| **U2** | Raspberry Pi Pico | RP2040 MCU — main controller | `RPi_Pico:RPi_Pico_SMD_TH` |
| **IC1** | SN74LVC1T45DBVR | 1-bit bidirectional voltage-level translator (3.3V↔5V) | `SN74LVC1T45DBVR:SOT95P280X145-6N` |
| **IC2** | SN74LVC1T45DBVR | 1-bit bidirectional voltage-level translator (3.3V↔5V) | `SN74LVC1T45DBVR:SOT95P280X145-6N` |
| **IC3** | TXU0304QPWRQ1 | 4-bit unidirectional voltage-level translator, A→B, 1.65–5.5V | `TXU0304QPWRQ1:TXU0304QPWRQ1` (TSSOP-14) |
| **J1** | Edge connector (2×8) | 16-pin PCB edge connector to cartridge board | `EdgeCon:Edge_connector_02x08` |
| **J2** | QL_CONN (2×7 pin header) | 14-pin header to Sinclair QL MicroDrive bus | `Connector_PinHeader_2.54mm:PinHeader_2x07_P2.54mm_Vertical` |
| **R1** | 4.7 kΩ | Pull-up/pull-down resistor | `Resistor_SMD:R_0603_1608Metric` |
| **C1–C6** | 100 nF | Decoupling capacitors | `Capacitor_SMD:C_0603_1608Metric` |

### 1.3 Raspberry Pi Pico (U2) Pin Assignment

The Pico's GPIOs are grouped by function. Net labels connect them to the surrounding circuits.

| Pico Pin | GPIO | Net Label | Direction | Function |
|---|---|---|---|---|
| 1 | GPIO0 | *(connected)* | — | — |
| 2 | GPIO1 | *(connected)* | — | — |
| 4 | GPIO2 | — | — | — |
| 5 | GPIO3 | — | — | — |
| 6 | GPIO4 | — | — | — |
| 7 | GPIO5 | — | — | — |
| 9 | GPIO6 | — | — | — |
| 10 | GPIO7 | — | — | — |
| 11 | GPIO8 | NC | — | Not connected |
| 12 | GPIO9 | NC | — | Not connected |
| 14 | GPIO10 | NC | — | Not connected |
| 15 | GPIO11 | NC | — | Not connected |
| 16 | GPIO12 | NC | — | Not connected |
| 17 | GPIO13 | NC | — | Not connected |
| 19 | GPIO14 | *(connected)* | — | — |
| 20 | GPIO15 | *(connected)* | — | — |
| 21 | GPIO16 | PICO_CS_IN | In | QL Cartridge Select input (from QL via level-shifter) |
| 22 | GPIO17 | PICO_CS_OUT | Out | QL Cartridge Select output |
| 24 | GPIO18 | *(connected)* | — | — |
| 25 | GPIO19 | *(connected)* | — | — |
| 26 | GPIO20 | *(connected)* | — | — |
| 27 | GPIO21 | *(connected)* | — | — |
| 29 | GPIO22 | *(connected)* | — | — |
| 31 | GPIO26_ADC0 | *(connected)* | — | — |
| 32 | GPIO27_ADC1 | *(connected)* | — | — |
| 34 | GPIO28_ADC2 | *(connected)* | — | — |
| 36 | 3V3 | +3V3 | Power out | 3.3V output to level-shifter VCCA |
| 39 | VSYS | +5V | Power in | 5V input from LM7805 |
| 40 | VBUS | +5V | Power | USB 5V pass-through |

> **Note:** Several GPIO pins in the NC/no-connect group (GPIO8–GPIO13) have explicit no-connect markers in the schematic. Exact GPIO-to-function mapping for the remaining connected pins is fully defined in the firmware (`hw.h`, `PIO_machines.pio.h`).

### 1.4 Voltage Level Translation

The Sinclair QL's MicroDrive bus operates at 5V logic while the Pico operates at 3.3V. Three ICs handle the translation:

**IC1 & IC2 — SN74LVC1T45DBVR** (Texas Instruments, 6-pin SOT-23)
- Single-bit, **bidirectional** translator
- VCCA = 3.3V (Pico side), VCCB = 5V (QL side)
- DIR pin selects transfer direction
- IC1 handles one QL signal line; IC2 handles another
- NET: `HEAD_DIR` controls the DIR pin of these translators

**IC3 — TXU0304QPWRQ1** (Texas Instruments, TSSOP-14)
- 4-channel, **unidirectional** A→B translator (3.3V→5V)
- VCCA = 3.3V (Pico side), VCCB = 5V (QL side)
- OE pin enabled
- Translates 4 Pico output signals to 5V QL bus signals

### 1.5 Sinclair QL MicroDrive Bus Interface (J2 — QL_CONN)

**J2** is a 2×7 pin header (14 pins) that connects to the Sinclair QL's internal MicroDrive bus.

| Net Label | Direction (Pico) | QL Signal Function |
|---|---|---|
| `QL_CS_IN` | Input | Cartridge Select input — QL selects this drive unit |
| `QL_CS_OUT` | Output | Cartridge Select output — daisy-chained to next drive |
| `QL_CS_CK` | Input | Cartridge Select Clock |
| `QL_HEAD1` | Bidirectional | Read/Write Head 1 data |
| `QL_HEAD2` | Bidirectional | Read/Write Head 2 data |
| `QL_ERASE` | Output | Erase head drive signal |
| `QL_R/~W` | Input | Read/Write direction signal (HIGH=Read, LOW=Write) |
| GND | — | Ground reference |
| +9V | — | 9V from QL supply |

The Pico-side equivalents carry the `PICO_` prefix:
`PICO_CS_IN`, `PICO_CS_OUT`, `PICO_CS_CK`, `PICO_HEAD1`, `PICO_HEAD2`, `PICO_ERASE`, `PICO_R/~W`

### 1.6 Cartridge Board Interface (J1 — Edge Connector)

**J1** is a 16-pin PCB edge connector (`EdgeCon:Edge_connector_02x08`) that mates with the cartridge board's J1. Signals are prefixed `UI_` and carry the user-interface and cartridge peripherals.

| J1 Pin | Net Label | Direction | Function |
|---|---|---|---|
| 1 | `UI_BTN_SELECT` | Input | SELECT button from cartridge |
| 2 | `UI_DETECT` | Input | SD card presence detect |
| 3 | `UI_BTN_NEXT` | Input | NEXT button from cartridge |
| 4 | `UI_SD_MOSI` | Output | SPI MOSI to SD card |
| 5 | `UI_BTN_BACK` | Input | BACK button from cartridge |
| 6 | `UI_SD_MISO` | Input | SPI MISO from SD card |
| 7 | `UI_SCR_SCL` | Output | I²C SCL to OLED screen |
| 8 | `UI_SD_SCK` | Output | SPI clock to SD card |
| 9 | `UI_SCR_SDA` | Bidirectional | I²C SDA to OLED screen |
| 10 | `UI_SD_CS` | Output | SPI chip-select to SD card |
| 11 | GND | — | Ground |
| 12 | `UI_LD_WRITE` | Output | WRITE LED drive |
| 13 | `UI_LD_READ` | Output | READ LED drive |
| 14 | `UI_LD_WRITE` | Output | WRITE LED (duplicate path) |
| 15 | `UI_LD_SELECT` | Output | SELECT LED drive |
| 16 | `UI_LD_READ` | Output | READ LED (duplicate path) |

> **Note:** Pin ordering above is derived from the net labels visible on each wire segment connecting to J1. Some signals appear on paired pins (read and write LEDs are driven redundantly across two edge connector pads for reliability). Exact pin-to-net mapping should be confirmed against the PCB layout.

---

## 2. MicroPicoDriveCartridge — Cartridge Board

### 2.1 Overview

The cartridge board is a plug-in module that provides:
- MicroSD card storage (SPI interface)
- Small OLED display (I²C interface)
- Three navigation buttons (BACK, NEXT, SELECT)
- Three status LEDs (READ, WRITE, SELECT), each with 1 kΩ current-limiting resistor
- 16-pin PCB edge connector to the main board

**Power:** Supplied entirely from the main board via J1 — only **+3.3V** and **GND** are used.

### 2.2 Component Bill of Materials

| Ref | Value / Part | Description | Footprint |
|---|---|---|---|
| **SD1** | SDCard | MicroSD card slot (SPI mode) | `CartridgeComponents:SDCard` |
| **SCR1** | Screen | OLED/LCD display (I²C, 4-pin connector) | `CartridgeComponents:Screen` |
| **SW1** | SW_Push | Pushbutton — BACK | `Button_Switch_SMD:SW_Push_1P1T_NO_6x6mm_H9.5mm` |
| **SW2** | SW_Push | Pushbutton — NEXT | `Button_Switch_SMD:SW_Push_1P1T_NO_6x6mm_H9.5mm` |
| **SW3** | SW_Push | Pushbutton — SELECT | `Button_Switch_SMD:SW_Push_1P1T_NO_6x6mm_H9.5mm` |
| **D1** | LED | Read activity LED (3mm THT) | `LED_THT:LED_D3.0mm` |
| **D2** | LED | Write activity LED (3mm THT) | `LED_THT:LED_D3.0mm` |
| **D3** | LED | Select/status LED (3mm THT) | `LED_THT:LED_D3.0mm` |
| **R1** | 1 kΩ | Current-limiting resistor for D1 | `Resistor_SMD:R_0603_1608Metric` |
| **R2** | 1 kΩ | Current-limiting resistor for D2 | `Resistor_SMD:R_0603_1608Metric` |
| **R3** | 1 kΩ | Current-limiting resistor for D3 | `Resistor_SMD:R_0603_1608Metric` |
| **J1** | Conn_02x08_Odd_Even | 16-pin PCB edge connector to main board | `CartridgeComponents:Edge` |

### 2.3 SD Card Interface (SD1)

SPI mode. All signals route through J1 to the Pico on the main board.

| SD1 Pin | Name | Net Label | Notes |
|---|---|---|---|
| 1 | DETECT | `UI_DETECT` | Card present detection (active LOW typical) |
| 2 | D1 | NC | SPI mode — not used |
| 3 | D0/MISO | `UI_SD_MISO` | SPI data out from SD card |
| 4 | GND | GND | Ground |
| 5 | CLK | `UI_SD_SCK` | SPI clock |
| 6 | VCC | +3V3 | 3.3V supply |
| 7 | CMD/MOSI | `UI_SD_MOSI` | SPI data in to SD card |
| 8 | D3/CS | `UI_SD_CS` | SPI chip select (active LOW) |
| 9 | D2 | NC | SPI mode — not used |

### 2.4 OLED Screen Interface (SCR1)

I²C interface, 4-pin connector.

| SCR1 Pin | Name | Net Label |
|---|---|---|
| 1 | GND | GND |
| 2 | VCC | +3V3 |
| 3 | SCL | `UI_SCR_SCL` |
| 4 | SDA | `UI_SCR_SDA` |

Firmware uses `ssd1306` driver (`ssd1306.c`). The ST7735 TFT driver is used on the main board side for a different display; the cartridge uses only the I²C OLED.

### 2.5 Navigation Buttons (SW1, SW2, SW3)

All buttons are normally-open SMD pushbuttons (6×6 mm, 9.5 mm height). One terminal connects to the signal net; the other connects to GND. The Pico reads these active-low signals.

| Ref | Net Label | Function |
|---|---|---|
| SW1 | `UI_BTN_BACK` | Navigate back / cancel |
| SW2 | `UI_BTN_NEXT` | Navigate forward / next item |
| SW3 | `UI_BTN_SELECT` | Confirm / select item |

### 2.6 Status LEDs (D1, D2, D3)

3mm THT LEDs with 1 kΩ series resistors (R1, R2, R3). Driven from the Pico via the edge connector.

| Ref | Net Label (anode side via resistor) | Function |
|---|---|---|
| D1 | `UI_LD_READ` | Lit during read operations |
| D2 | `UI_LD_WRITE` | Lit during write operations |
| D3 | `UI_LD_SELECT` | Lit to indicate active/selected drive |

LED current (assuming 3.3V rail, ~2V Vf): I = (3.3 − 2.0) / 1000 ≈ **1.3 mA** — suitable for low-brightness status indication.

### 2.7 Cartridge Edge Connector (J1)

J1 (`CartridgeComponents:Edge`) is the 16-pin PCB edge connector (2×8, odd/even numbering) that plugs into the main board's J1.

| J1 Pin | Net Label | Direction (from cartridge) | Function |
|---|---|---|---|
| 1 | `UI_LD_READ` | Input | READ LED drive from main board |
| 2 | `UI_LD_READ` | Input | READ LED drive (redundant pad) |
| 3 | GND | — | Ground |
| 4 | `UI_LD_WRITE` | Input | WRITE LED drive from main board |
| 5 | `UI_SCR_SDA` | Bidirectional | I²C SDA to OLED |
| 6 | `UI_SD_CS` | Input | SPI CS from Pico |
| 7 | `UI_SCR_SCL` | Input | I²C SCL from Pico |
| 8 | `UI_SD_SCK` | Input | SPI clock from Pico |
| 9 | `UI_BTN_BACK` | Output | BACK button signal to Pico |
| 10 | `UI_SD_MISO` | Output | SPI MISO from SD card to Pico |
| 11 | `UI_BTN_NEXT` | Output | NEXT button signal to Pico |
| 12 | `UI_SD_MOSI` | Input | SPI MOSI from Pico to SD card |
| 13 | `UI_BTN_SELECT` | Output | SELECT button signal to Pico |
| 14 | `UI_DETECT` | Output | SD card detect to Pico |
| 15 | `UI_LD_SELECT` | Input | SELECT LED drive from main board |
| 16 | GND | — | Ground |

---

## 3. Signal Cross-Reference: Main Board ↔ Cartridge

All inter-board signals use the `UI_` prefix. The same net name appears on both schematics and is physically connected through the edge connector.

| Net Label | Main Board origin (Pico GPIO) | Cartridge destination | Type |
|---|---|---|---|
| `UI_SD_MISO` | Pico SPI RX | SD1 pin 3 (D0/MISO) | SPI data |
| `UI_SD_MOSI` | Pico SPI TX | SD1 pin 7 (CMD/MOSI) | SPI data |
| `UI_SD_SCK` | Pico SPI SCK | SD1 pin 5 (CLK) | SPI clock |
| `UI_SD_CS` | Pico GPIO | SD1 pin 8 (D3/CS) | SPI CS |
| `UI_DETECT` | Pico GPIO (input) | SD1 pin 1 (DETECT) | Card detect |
| `UI_SCR_SCL` | Pico I²C SCL | SCR1 pin 3 | I²C clock |
| `UI_SCR_SDA` | Pico I²C SDA | SCR1 pin 4 | I²C data |
| `UI_BTN_BACK` | Pico GPIO (input) | SW1 | Button |
| `UI_BTN_NEXT` | Pico GPIO (input) | SW2 | Button |
| `UI_BTN_SELECT` | Pico GPIO (input) | SW3 | Button |
| `UI_LD_READ` | Pico GPIO (output) | D1 via R1 | LED |
| `UI_LD_WRITE` | Pico GPIO (output) | D2 via R2 | LED |
| `UI_LD_SELECT` | Pico GPIO (output) | D3 via R3 | LED |

---

## 4. Custom Symbol Libraries

### CartridgeComponents.kicad_sym
Located: `Hardware/MicroPicoDriveCartridge/MicroPicoDriveCartridge/`

| Symbol | Pins | Description |
|---|---|---|
| `SDCard` | 9 | MicroSD slot: VCC, GND, CLK, CMD/MOSI, D0/MISO, D1, D2, D3/CS, DETECT |
| `Screen` | 4 | Generic I²C display: VCC, GND, SCL, SDA |

### RPi_Pico.kicad_sym
Located: `Hardware/MicroPicoDrive/`

| Symbol | Pins | Description |
|---|---|---|
| `Pico` | 40 | Raspberry Pi Pico: GPIO0–GPIO28, GND×7, VSYS, VBUS, 3V3, 3V3_EN, RUN, ADC_VREF, AGND |

---

## 5. Custom Footprint Libraries

| Library folder | Contents |
|---|---|
| `EdgeCon.pretty/` | `Edge_connector_02x08.kicad_mod` — 2×8 PCB edge connector (main board J1) |
| `RPi_Pico.pretty/` | `RPi_Pico_SMD_TH.kicad_mod`, USB Micro-B, audio jack (CUI SJ-3523), DB-15, MBR120 SOD-123 diode |
| `SN74LVC1T45DBVR.pretty/` | `SOT95P280X145-6N.kicad_mod` — SOT-23-6 footprint for IC1, IC2 |
| `TXU0304QPWRQ1.pretty/` | `TXU0304QPWRQ1.kicad_mod` — TSSOP-14 footprint for IC3 |
| `CartridgeComponents.pretty/` | `Edge.kicad_mod` (cartridge J1), `Screen.kicad_mod`, `SDCard.kicad_mod` |

---

## 6. Design Notes and Constraints

1. **Voltage levels:** The QL MicroDrive bus is 5V TTL. The Pico is strictly 3.3V. All QL-facing bidirectional signals pass through IC1 or IC2 (SN74LVC1T45DBVR). The four unidirectional Pico→QL output lines use IC3 (TXU0304QPWRQ1).

2. **HEAD_DIR net:** This signal drives the DIR pin of IC1 and IC2, selecting whether the current operation is a read (Pico receives) or a write (Pico drives). It must be set by firmware before any head data transfer.

3. **Power sequencing:** The LM7805 output (+5V) powers the level-shifter VCCB and Pico VSYS. The Pico's internal regulator then provides +3.3V to level-shifter VCCA. No special sequencing is needed — the Pico's +3.3V output rises after VSYS, which is the correct order for the level-shifters.

4. **SD card power:** The SD card on the cartridge is powered directly from +3.3V with no switching. If SD card hot-swap is desired in future revisions, a load-switch on the +3.3V rail would be needed.

5. **I²C pull-ups:** Not explicitly shown on the cartridge schematic for SCL/SDA. The Pico's internal pull-ups or pull-ups on the main board should be confirmed in firmware and PCB review.

6. **Button debounce:** No hardware RC debounce shown; handled in firmware (UserInterface.c).

7. **No-connect pins:** GPIO8–GPIO13 on the Pico have explicit no-connect markers. These are available for future expansion.

8. **DB-15 connector footprint** (`DSUB-15-L77HDE15SD1CH4F.kicad_mod`) exists in `RPi_Pico.pretty/` but is not instantiated in the current schematic — likely a placeholder or legacy artifact.

---

## 7. Related Files

| File | Purpose |
|---|---|
| `MicroPicoDrive.kicad_pcb` | PCB layout for main board |
| `MicroPicoDriveCartridge.kicad_pcb` | PCB layout for cartridge board |
| `Firmware/hw.h` | GPIO pin-number assignments |
| `Firmware/PIO_machines.pio.h` | PIO programs for MicroDrive head data timing |
| `Firmware/MicroDriveControl.c` | QL MicroDrive protocol emulation |
| `Firmware/UserInterface.c` | Button, LED, and screen handling |
| `Firmware/diskio.c` | SD card SPI disk I/O |
| `Firmware/ST7735_TFT.c` | TFT display driver (main board) |
| `Firmware/ssd1306.c` | OLED I²C display driver (cartridge) |
