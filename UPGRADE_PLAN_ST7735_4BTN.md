# Upgrade Plan: Replace Cartridge I²C OLED with SPI ST7735S + 4-Button Module + Activity LED

**Module:** 0.96" 80×160 IPS, ST7735S driver, 4-wire SPI, 65K colour, with 4 integrated pushbuttons (K1–K4)

**Module connector pins (left 8-pin header):**
`GND | VCC | SCL(clock) | SDA(MOSI) | RES(reset) | DO(D/C) | CS | BLK(backlight)`

**Module button pins (right side):** `K1 | K2 | K3 | K4`

---

## Why the 16-pin edge connector fits exactly

| Budget item | J1 Pins |
|---|---|
| Power: +3V3 (pin 1) + GND (pin 3) | 2 |
| SD card: MISO, MOSI, SCK, CS (pins 10, 12, 8, 6) | 4 |
| Cartridge-detect UI_DETECT → GND pull (pin 14) | 1 |
| SPI display: CS, DC, MOSI, SCK (pins 2, 4, 5, 7) | 4 |
| Buttons K1–K4 (pins 9, 11, 13, 15) | 4 |
| LED_ACTIVITY (pin 16) | 1 |
| **Total** | **16** |

**Two design choices that eliminate two pins:**
- **RES (reset):** passive RC circuit on cartridge PCB (10 kΩ to +3V3, 100 nF to GND). Generates an automatic ≥1 ms LOW pulse at power-on. No GPIO or J1 pin needed. See *RC Reset Circuit* section.
- **BLK (backlight):** tied directly to +3V3 on cartridge PCB (always-on). No GPIO needed.

---

## J1 Edge-Connector Pin Assignment Changes

> J1 is a 2×8 PCB edge connector. Odd pins = left side (top→bottom: 1, 3, 5 … 15); even pins = right side (top→bottom: 2, 4, 6 … 16).
> Both boards must match pin-for-pin.
> Pin layout verified against the KiCad schematic screenshot: **Pin 1 = top-left, Pin 16 = bottom-right.**

| Pin | Side | **Current net** | **New net** | Notes |
|---|---|---|---|---|
| 1 | Left | +3V3 | +3V3 | unchanged |
| **2** | Right | `UI_LD_READ` | **`UI_SCR_CS`** | LED Read → display CS |
| 3 | Left | GND | GND | unchanged |
| **4** | Right | `UI_LD_WRITE` | **`UI_SCR_DC`** | LED Write → display D/C |
| **5** | Left | `UI_SCR_SDA` | **`UI_SCR_MOSI`** | I²C SDA → SPI1 MOSI |
| 6 | Right | `UI_SD_CS` | `UI_SD_CS` | unchanged |
| **7** | Left | `UI_SCR_SCL` | **`UI_SCR_SCK`** | I²C SCL → SPI1 clock |
| 8 | Right | `UI_SD_SCK` | `UI_SD_SCK` | unchanged |
| 9 | Left | `UI_BTN_BACK` | `UI_BTN_BACK` | unchanged (K1) |
| 10 | Right | `UI_SD_MISO` | `UI_SD_MISO` | unchanged |
| 11 | Left | `UI_BTN_NEXT` | `UI_BTN_NEXT` | unchanged (K2) |
| 12 | Right | `UI_SD_MOSI` | `UI_SD_MOSI` | unchanged |
| 13 | Left | `UI_BTN_SELECT` | `UI_BTN_SELECT` | unchanged (K3) |
| 14 | Right | `UI_DETECT` → GND | `UI_DETECT` → GND | unchanged — cartridge GND pull-down; GPIO 15 reads LOW = present |
| **15** | Left | `UI_LD_SELECT` | **`UI_BTN_K4`** | LED Select → 4th button |
| **16** | Right | NC | **`UI_LD_ACTIVITY`** | **new wire + label — blinking activity LED** |

**6 pins change (2, 4, 5, 7, 15, 16). 10 pins unchanged.**

---

## GPIO Changes (Pico RP2040)

| GPIO | Pico Pin | Current net / function | New net / function | Direction |
|---|---|---|---|---|
| 9 | 12 | `UI_LD_SELECT` / PIN_LED_SELECT | `UI_BTN_K4` — 4th button (pull-up) | OUT → IN |
| **10** | **14** | `UI_LD_READ` / PIN_LED_WRITE | **`UI_LD_ACTIVITY`** — activity LED blink | OUT (keep) |
| 11 | 15 | `UI_LD_WRITE` / PIN_LED_READ | not used — add NC marker | OUT → NC |
| 20 | 26 | `UI_SCR_SDA` (I²C SDA) | `UI_SCR_CS` — display chip select | OUT (keep) |
| 21 | 27 | `UI_SCR_SCL` (I²C SCL) | `UI_SCR_DC` — display D/C | OUT (keep) |
| 26 | 31 | NC in schematic | `UI_SCR_SCK` — SPI1 clock | NC → OUT |
| 27 | 32 | NC in schematic | `UI_SCR_MOSI` — SPI1 MOSI | NC → OUT |
| 12 | 16 | `UI_BTN_BACK` / PIN_BTN_BACK | K1 — same function | IN (unchanged) |
| 13 | 17 | `UI_BTN_NEXT` / PIN_BTN_NEXT | K2 — same function | IN (unchanged) |
| 14 | 19 | `UI_BTN_SELECT` / PIN_BTN_SELECT | K3 — same function | IN (unchanged) |
| 15 | 20 | `UI_DETECT` / PIN_UI_DETECT | unchanged | IN (unchanged) |

> **Note:** The firmware (`lib-st7735/include/hw.h`) already defines GPIO 20/21/26/27 as SPI1 TFT pins. The schematic shows GPIO 20/21 wired to the old I²C labels; GPIO 26/27 are NC. Schematic must be updated to match.

---

## Schematic Changes Required

### A. CartridgeComponents.kicad_sym — add new symbol

Add a `ST7735_SPI` symbol with 12 pins:

| Pin # | Name | Side | Type |
|---|---|---|---|
| 1 | GND | Left | Passive |
| 2 | VCC | Left | Passive |
| 3 | SCL | Left | Input |
| 4 | SDA | Left | Input |
| 5 | RES | Left | Input |
| 6 | DO | Left | Input (D/C line) |
| 7 | CS | Left | Input |
| 8 | BLK | Left | Input |
| 9 | K1 | Right | Passive |
| 10 | K2 | Right | Passive |
| 11 | K3 | Right | Passive |
| 12 | K4 | Right | Passive |

Reference prefix: `SCR`, Value: `ST7735_SPI`

---

### B. MicroPicoDriveCartridge.kicad_sch

**Remove these components and their wires:**
- D1, D2, D3 — LEDs (`LED_THT:LED_D3.0mm`)
- R1, R2, R3 — 1 kΩ LED resistors (`Resistor_SMD:R_0603_1608Metric`)
- SW1, SW2, SW3 — pushbuttons (`Button_Switch_SMD:SW_Push_1P1T_NO_6x6mm_H9.5mm`)
- SCR1 — old I²C Screen symbol (`CartridgeComponents:Screen`)
- Dangling net labels after removal: `UI_LD_READ`, `UI_LD_WRITE`, `UI_LD_SELECT`, `UI_SCR_SCL`, `UI_SCR_SDA`

**Add ST7735 module symbol (new SCR1):**
- Place `CartridgeComponents:ST7735_SPI` as `SCR1`
- GND → GND power symbol
- VCC → +3V3 power symbol
- SCL → net label `UI_SCR_SCK`
- SDA → net label `UI_SCR_MOSI`
- RES → RC reset node (see below)
- DO → net label `UI_SCR_DC`
- CS → net label `UI_SCR_CS`
- BLK → +3V3 power symbol (always-on backlight)
- K1 → net label `UI_BTN_BACK`
- K2 → net label `UI_BTN_NEXT`
- K3 → net label `UI_BTN_SELECT`
- K4 → net label `UI_BTN_K4`

**Add RC reset circuit:**
- R_RST (10 kΩ, 0603): pin 1 → +3V3, pin 2 → RES node
- C_RST (100 nF, 0603): pin 1 → RES node, pin 2 → GND
- Wire from RES node to SCR1 pin RES

**Add LED_ACTIVITY circuit on J1 pin 16:**
- R_ACT (1 kΩ, 0603): pin 1 → net label `UI_LD_ACTIVITY`, pin 2 → D_ACT anode
- D_ACT (LED): cathode → GND
- Net label `UI_LD_ACTIVITY` also connects to J1 pin 16 — **draw wire from J1 pin 16 to this label (pin 16 is currently NC — a new wire must be added)**

**Update J1 wiring — 6 pins change:**

| J1 pin | Action | Old label | New label |
|---|---|---|---|
| 2 (right) | rename | `UI_LD_READ` | `UI_SCR_CS` |
| 4 (right) | rename | `UI_LD_WRITE` | `UI_SCR_DC` |
| 5 (left) | rename | `UI_SCR_SDA` | `UI_SCR_MOSI` |
| 7 (left) | rename | `UI_SCR_SCL` | `UI_SCR_SCK` |
| 15 (left) | rename | `UI_LD_SELECT` | `UI_BTN_K4` |
| 16 (right) | **add wire + label** | *(no wire — NC)* | `UI_LD_ACTIVITY` |

---

### C. MicroPicoDrive.kicad_sch (main board)

**Remove / disconnect from J1 and Pico:**
- `UI_LD_READ` at J1 pin 2 — disconnect from Pico GPIO 10 (pin 14)
- `UI_LD_WRITE` at J1 pin 4 — disconnect from Pico GPIO 11 (pin 15)
- `UI_LD_SELECT` at J1 pin 15 — disconnect from Pico GPIO 9 (pin 12)
- `UI_SCR_SDA` at J1 pin 5 — disconnect from Pico GPIO 20 (pin 26)
- `UI_SCR_SCL` at J1 pin 7 — disconnect from Pico GPIO 21 (pin 27)

**Add / reconnect at J1 and Pico:**

| J1 pin | New net | Pico GPIO | Pico pin | Action |
|---|---|---|---|---|
| 2 | `UI_SCR_CS` | GPIO 20 | 26 | GPIO 20 moves from J1 pin 5 to J1 pin 2 |
| 4 | `UI_SCR_DC` | GPIO 21 | 27 | GPIO 21 moves from J1 pin 7 to J1 pin 4 |
| 5 | `UI_SCR_MOSI` | GPIO 27 | 32 | new connection (GPIO 27 was NC) |
| 7 | `UI_SCR_SCK` | GPIO 26 | 31 | new connection (GPIO 26 was NC) |
| 15 | `UI_BTN_K4` | GPIO 9 | 12 | rename only |
| **16** | **`UI_LD_ACTIVITY`** | **GPIO 10** | **14** | **add wire from J1 pin 16 to GPIO 10** |

**Pico left-side GPIO updates:**
| GPIO | Pico pin | Action |
|---|---|---|
| 9 | 12 | Rename net label `UI_LD_SELECT` → `UI_BTN_K4` |
| 10 | 14 | Rename net label `UI_LD_READ` → `UI_LD_ACTIVITY`; re-route wire to J1 pin 16 |
| 11 | 15 | Remove net label `UI_LD_WRITE`; add no-connect marker |

**Pico right-side GPIO updates:**
| GPIO | Pico pin | Action |
|---|---|---|
| 20 | 26 | Rename `UI_SCR_SDA` → `UI_SCR_CS`; re-route wire to J1 pin 2 |
| 21 | 27 | Rename `UI_SCR_SCL` → `UI_SCR_DC`; re-route wire to J1 pin 4 |
| 26 | 31 | Remove NC marker; add net label `UI_SCR_SCK`; wire to J1 pin 7 |
| 27 | 32 | Remove NC marker; add net label `UI_SCR_MOSI`; wire to J1 pin 5 |

---

## PCB Changes Required

### MicroPicoDriveCartridge PCB

1. **Remove** footprints and copper for: D1, D2, D3, R1, R2, R3, SW1, SW2, SW3, SCR1
2. **Add** footprint for ST7735_SPI module (8-pin 2.54mm header + 4 button pads)
3. **Add** RC reset circuit: 0603 10 kΩ resistor + 0603 100 nF capacitor
4. **Add** LED_ACTIVITY circuit: 0603 1 kΩ resistor + LED, connected to J1 pad 16
5. **Re-route** J1 edge-connector pads for the 6 changed signals (pins 2, 4, 5, 7, 15, 16)

### MicroPicoDrive PCB (main board)

1. **Remove** traces: GPIO 10 → J1 pin 2 (`UI_LD_READ`) and GPIO 11 → J1 pin 4 (`UI_LD_WRITE`)
2. **Re-route / add** traces:
   - GPIO 20 (Pico pin 26) → J1 pin 2 (`UI_SCR_CS`)
   - GPIO 21 (Pico pin 27) → J1 pin 4 (`UI_SCR_DC`)
   - GPIO 26 (Pico pin 31) → J1 pin 7 (`UI_SCR_SCK`)
   - GPIO 27 (Pico pin 32) → J1 pin 5 (`UI_SCR_MOSI`)
   - GPIO 9 (Pico pin 12) → J1 pin 15 (`UI_BTN_K4`) — re-route from old LED_SELECT trace
   - **GPIO 10 (Pico pin 14) → J1 pin 16 (`UI_LD_ACTIVITY`) — new trace**

---

## Firmware Changes Required

### `Firmware/UserInterface.h`

```c
// REMOVE:
#define PIN_LED_SELECT  9
#define PIN_LED_WRITE   10
#define PIN_LED_READ    11

// ADD:
#define PIN_BTN_K4       9   // GPIO 9  — was PIN_LED_SELECT (J1 pin 15)
#define PIN_LED_ACTIVITY 10  // GPIO 10 — was PIN_LED_WRITE  (J1 pin 2)
// GPIO 11 (was PIN_LED_READ, J1 pin 4) — freed, no define needed
```

### `Firmware/UserInterface.c`

**1. `init_leds()` — replace three LED inits with single activity LED:**
```c
void init_leds()
{
    gpio_init(PIN_LED_ON);
    gpio_init(PIN_LED_ACTIVITY);

    gpio_set_dir(PIN_LED_ON, true);
    gpio_set_dir(PIN_LED_ACTIVITY, true);

    gpio_put(PIN_LED_ON, 1);
    gpio_put(PIN_LED_ACTIVITY, 0);
}
```

**2. `init_buttons()` — add K4 pull-up:**
```c
gpio_init(PIN_BTN_K4);
gpio_set_dir(PIN_BTN_K4, false);
gpio_pull_up(PIN_BTN_K4);
```

**3. `process_md_to_ui_event()` — replace three separate LED cases with unified ACTIVITY LED:**

Key rule: the LED is toggled **only** by `BUFFERSET` events, never by `READING`/`WRITTING` state entry. This guarantees it always blinks during activity and never goes solid, even under constant back-to-back operations.

```c
case MTU_MD_DESELECTED:
    mdInUse = false;
    inFormat = false;
    LED_OFF(PIN_LED_ACTIVITY);     // explicit off on idle
    break;

case MTU_MD_SELECTED:
    mdInUse = true;
    break;

case MTU_MD_READING:               // no LED change — BUFFERSET events drive blinking
case MTU_MD_WRITTING:              // no LED change — BUFFERSET events drive blinking
    break;

case MTU_BUFFERSET_READ:
    process_md_read(evt->arg);
    gpio_put(PIN_LED_ACTIVITY, !gpio_get(PIN_LED_ACTIVITY)); // toggle per sector
    break;

case MTU_BUFFERSET_WRITTEN:
    process_md_write(evt->arg);
    gpio_put(PIN_LED_ACTIVITY, !gpio_get(PIN_LED_ACTIVITY)); // toggle per sector
    break;
```

LED_ACTIVITY states:
- **Idle / deselected:** OFF
- **Active (sectors transferring):** toggles on every sector — always blinks, never solid
- **Selected but idle (between operations):** holds last toggle state until deselect

**4. `CARTRIDGE_READY` state — add K4 as secondary eject trigger:**
```c
if (BUTTON_PRESSED(PIN_BTN_BACK) || BUTTON_PRESSED(PIN_BTN_K4))
{
    debounce_button(BUTTON_PRESSED(PIN_BTN_BACK) ? PIN_BTN_BACK : PIN_BTN_K4);
    rewind_path();
    uiState = OPEN_FOLDER;
    ...
```

---

## Module Wiring Reference

```
ST7735S Module (left 8-pin header)      Pico GPIO / Net         J1 pin
────────────────────────────────────    ──────────────────────  ──────
GND                                  →  GND                     pin 3
VCC (3.3V)                           →  +3V3                    pin 1
SCL (SPI clock)                      →  GPIO 26 / UI_SCR_SCK    pin 7
SDA (MOSI)                           →  GPIO 27 / UI_SCR_MOSI   pin 5
RES (reset)                          →  RC circuit on cartridge —
DO  (D/C — data/command)             →  GPIO 21 / UI_SCR_DC     pin 4
CS  (chip select)                    →  GPIO 20 / UI_SCR_CS     pin 2
BLK (backlight)                      →  +3V3 (tied on cartridge) —

ST7735S Module (right button pads)      Pico GPIO / Net         J1 pin
────────────────────────────────────    ──────────────────────  ──────
K1                                   →  GPIO 12 / UI_BTN_BACK   pin 9
K2                                   →  GPIO 13 / UI_BTN_NEXT   pin 11
K3                                   →  GPIO 14 / UI_BTN_SELECT pin 13
K4                                   →  GPIO  9 / UI_BTN_K4     pin 15

LED_ACTIVITY circuit (on cartridge)     Pico GPIO / Net         J1 pin
────────────────────────────────────    ──────────────────────  ──────
1 kΩ resistor + LED to GND           →  GPIO 10 / UI_LD_ACTIVITY pin 16
```

---

## RC Reset Circuit (on cartridge PCB)

The ST7735S RES pin requires a LOW pulse at power-on. A passive RC circuit generates this automatically, using no GPIO and no J1 pin.

### Circuit

```
        +3V3
          |
         [R]  10 kΩ (0603)
          |
          +-----------> RES (SCR1 pin 5)
          |
         [C]  100 nF (0603)
          |
         GND
```

### How it works

1. **Power-off:** C is discharged → node = 0 V → RES is LOW → ST7735S held in reset.
2. **Power-on:** +3V3 rises; C charges through R. RES remains LOW during charging.
3. **Reset release:** Node follows V(t) = 3.3 × (1 − e^(−t/τ)), τ = 10 kΩ × 100 nF = **1 ms**.
   ST7735S logic-HIGH threshold ≈ 2.3 V, reached at t ≈ 1.2 ms.
4. **Steady state:** C fully charged, RES held at +3V3 for rest of session.

### Timing diagram

```
+3V3 ─────────────────────────────────
RES   ___/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
      |←~1.2ms→|
       (reset)   (running)
```

Place R and C as close as possible to the module's RES pad.
