# GameBoy 🎮
![Static Badge](https://img.shields.io/badge/status-ongoing-green)


A custom dual-processor embedded development platform built around the **Texas Instruments MSPM0G3507** and the **Espressif ESP32-S3-WROOM-1**, designed for IoT connectivity, sensor data collection, and general-purpose prototyping.

> Designed from scratch in KiCad 9.0.8 — schematic complete, PCB layout in progress.

## Project Summary

GameBoy is a dual-MCU embedded platform that combines the real-time control of the MSPM0G3507 with the wireless connectivity of the ESP32-S3-WROOM-1. The board uses a dedicated UART bridge between the two processors, includes onboard programming and debug interfaces, and is being developed as a compact hardware platform for prototyping, sensor integration, and IoT-oriented experiments. The current revision is a 4-layer board to support cleaner routing, better power integrity, and improved overall layout density.

---

## Overview

GameBoy is a compact, battery-capable development board that pairs the real-time processing power of the MSPM0G3507 with the WiFi and Bluetooth capabilities of the ESP32-S3, communicating over a dedicated UART bus. The goal is a flexible platform that can be adapted across a wide range of embedded applications without redesigning hardware each time.

This is not a hobbyist shield or a breakout board. Every subsystem — power architecture, programming interfaces, peripheral connections, and RF considerations — has been designed with deliberate engineering decisions documented below.

---

## Hardware Specifications

| Parameter | Details |
|---|---|
| Main MCU | TI MSPM0G3507SPTR — ARM Cortex-M0+, 80MHz, 128KB Flash, 32KB SRAM |
| Co-processor | Espressif ESP32-S3-WROOM-1 — Xtensa LX7, WiFi 802.11 b/g/n, BLE 5.0 |
| Inter-processor bus | UART (MSPM0 UART1 ↔ ESP32-S3 UART0) |
| Supply voltage | 5V via USB / Single-cell Li-ion (18650) |
| System voltage | 3.3V (SPX3819M5 LDO) |
| Battery charging | MCP73831 linear charger — 300mA via USB-C |
| Power path | LM66200DRLR ideal diode — automatic USB / battery switchover |
| Crystal | 8MHz external oscillator for MSPM0 |
| PCB layers | 4-layer |
| EDA tool | KiCad 9.0.8 |

---

## Block Diagram

```
                        ┌─────────────────────────────────────┐
                        │           Power Section             │
  USB-C (5V) ──────────►│ MCP73831 ──► 18650 Li-ion           │
                        │ LM66200 (auto power path switching)  │
                        │ SPX3819M5 LDO ──► 3.3V rail         │
                        └────────────────┬────────────────────┘
                                         │ 3.3V
                  ┌──────────────────────┼──────────────────────┐
                  │                      │                      │
         ┌────────▼────────┐    ┌────────▼────────┐   ┌────────▼────────┐
         │  MSPM0G3507     │    │  ESP32-S3        │   │  LIS2DH12       │
         │  Main processor │◄──►│  WiFi + BT       │   │  Accelerometer  │
         │  SWD via STLink │UART│  CP2102N + auto  │   │  SPI            │
         │  UART0 debug    │    │  boot circuit    │   └─────────────────┘
         └────────┬────────┘    └────────┬─────────┘
                  │                      │
         ┌────────▼────────┐    ┌────────▼─────────┐
         │  GPIO Bank A    │    │  GPIO Bank B      │
         │  MSPM0 pins     │    │  ESP32-S3 pins    │
         └─────────────────┘    └──────────────────┘
```

---

## Subsystems

### Main MCU — MSPM0G3507
- ARM Cortex-M0+ running at up to 80MHz
- Programmed via SWD using STLink V2 through a 4-pin header (SWCLK, SWDIO, NRST, GND)
- UART0 exposed via onboard CH340E for debug output
- 4 UART peripherals available — UART1 dedicated to ESP32-S3 communication
- Full GPIO bank broken out to 2.54mm headers

### Co-processor — ESP32-S3-WROOM-1
- Handles all WiFi and Bluetooth stack operations
- Programmed independently via dedicated USB-C port and CP2102N USB-UART bridge
- Auto-boot circuit (two MMBT2222A NPN transistors) — no manual button pressing required to flash
- Manual reset (SW1) and boot (SW2) buttons included as fallback
- IO19 and IO20 (native USB D+/D−) left available for future USB device applications
- GPIO bank broken out to 2.54mm headers

### Power Architecture
- **LM66200DRLR** ideal diode controller manages automatic switchover between USB-C (5V) and Li-ion battery (3.7V) — USB always takes priority when present
- **MCP73831** linear charger handles single-cell Li-ion charging at 300mA (set via 3.3K RPROG resistor), powered from the ESP32-S3 USB-C port
- **SPX3819M5** 500mA LDO regulates system 3.3V rail
- Two 18650 cells in parallel supported — no balancing circuit required for parallel configuration
- Battery voltage monitoring via MSPM0 ADC through resistor divider

### Programming Interfaces
| Target | Method | Connector |
|---|---|---|
| MSPM0G3507 firmware | STLink V2 via SWD | 4-pin header |
| MSPM0 debug UART | CH340E via USB Micro | USB Micro B |
| ESP32-S3 firmware | CP2102N + auto-boot via USB-C | USB-C |

### Peripheral — LIS2DH12 Accelerometer
- ST MEMS 3-axis accelerometer
- Connected via SPI to MSPM0G3507
- INT1 and INT2 interrupt lines routed to MSPM0 GPIO

---

## Repository Structure

```
GameBoy/
├── Schematic/
│   ├── V1.0.pdf          # Main MCU schematic sheet
│   ├── V1.0.pdf         # Main MCU schematic sheet + ESP32-S3 subcircuit sheet
├── Docs/
│   └── esp32_devkitc_v4-sch.pdf        # ESP32 Device schematic reference
|   └── mspm0g3507_Ti.pdf       # MSPM0 Instruction Manual
├── Docs/
│   ├── V1.0.md/                     # upadtes implemented in schematic one
│   └── V2.0.md/                     # updates implemented in schematic two
└── README.md
```

---

## Design Decisions

A few non-obvious decisions made during the design process, documented for anyone reading the schematic.

**Why MSPM0G3507 and not STM32?**
TI's MSPM0 series offers a competitive Cortex-M0+ core with a generous free tier on Code Composer Studio and strong documentation. The LQFP-48 package is hand-solderable and the peripheral set covers everything needed for this platform.

**Why ESP32-S3 and not classic ESP32?**
The S3 supports BLE 5.0 and has a native USB peripheral on IO19/IO20, keeping future options open. The WROOM-1 module handles the RF design, antenna matching, and FCC certification — removing the most complex part of ESP32 integration from the custom PCB.

**Why CP2102N for ESP32 programming instead of using native USB?**
Native USB on IO19/IO20 works, but is dependent on the ESP32-S3 firmware being functional. A dedicated CP2102N with auto-boot circuit means the programming interface is always independent of firmware state — critical on a prototyping board where firmware is frequently broken during development.

**Why 4-layer PCB?**
The 4-layer stackup improves power distribution, signal routing, and ground integrity for the dual-MCU design. It also gives cleaner separation between noisy digital sections, the ESP32-S3 RF area, and the power subsystem while making the board easier to route at the current feature density.

**Why LM66200 for power path instead of a discrete P-FET?**
The LM66200 is an ideal diode controller with 40mΩ on-resistance, automatic priority switching, reverse current blocking, and 1.32µA quiescent current — all in one IC with no external switching logic required. The discrete P-FET approach requires careful gate biasing and does not block reverse current.

---

## Current Status

| Milestone | Status |
|---|---|
| System architecture | ✅ Complete |
| MSPM0 schematic | ✅ Complete |
| ESP32-S3 schematic | ✅ Complete |
| Power section schematic | ✅ Complete |
| PCB layout | 🔄 In progress |
| Fabrication | ⏳ Pending layout |
| Firmware — MSPM0 | ⏳ Pending hardware |
| Firmware — ESP32-S3 | ⏳ Pending hardware |

---

## Planned Features (Future Revisions)

- Motor driver header with PWM breakout from MSPM0 timers
- QWIIC / Stemma QT I2C connector for plug-and-play sensor expansion
- USB-C PD negotiation for faster battery charging
- Onboard LiPo fuel gauge IC for accurate battery percentage reporting
- KiCad PCB layout with JLCPCB-compatible BOM and CPL files

---

## Tools Used

- **KiCad 9.0.8** — schematic capture and PCB layout
- **Code Composer Studio** — MSPM0 firmware development
- **ESP-IDF** — ESP32-S3 firmware development
- **LCSC** — component sourcing

---

## Author

Darkops-CPU (Jayant Yadav)

---

*The name GameBoy has no relation to Nintendo's product. It is an arbitrary project codename.*