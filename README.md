# HADES

![Static Badge](https://img.shields.io/badge/status-ongoing-green) 


**An embedded control brain for people who've outgrown the Arduino Uno.**

HADES is a dual-MCU prototyping and control board built around one idea: it is the *brain*, not the *muscle*. It runs your control loops and talks to the network — it does **not** carry motor current. Power for actuators lives on a separate, swappable **HADES shield**, so the brain stays clean, cool, and reusable across every robot you bolt it to.

> **Status:** 🚧 In active design. The architecture below is locked; hardware is in progress. Nothing here is manufactured yet — treat part-specific and pinout details marked _(TBD)_ as not-yet-final.

---

## Table of contents

- [Philosophy: brain, not muscle](#philosophy-brain-not-muscle)
- [Architecture](#architecture)
- [The HADES bus (the shield contract)](#the-hades-bus-the-shield-contract)
- [Planned features](#planned-features)
- [Power](#power)
- [Toolchain & programming](#toolchain--programming)
- [Repository structure](#repository-structure)
- [Getting started](#getting-started)
- [Roadmap](#roadmap)
- [Contributing & succession](#contributing--succession)
- [License](#license)
- [Origin](#origin)

---

## Philosophy: brain, not muscle

Most "do-everything" dev boards try to be the controller *and* the power stage on one PCB. That fights physics: high-current switching, motor back-EMF, heat, and ground bounce all land right next to the quiet analog front-end and the radio that make a good controller good.

HADES refuses that trade. The board sends control signals to the muscle and reads sensors back — nothing more. Actuator power is handled off-board, on a companion shield. The payoff:

- **Clean signals.** The analog front-end and the wireless radio live in a quiet neighborhood.
- **One brain, many muscles.** A small-servo shield today, a brushed-motor shield next term, a big BLDC shield later — same HADES underneath.
- **Failure isolation.** When someone stalls a motor and lets the smoke out of a driver, the cheap shield dies, not the brain.
- **Reusability.** The brain never needs a redesign to drive something new. That work moves to the shield, where it belongs.

## Architecture

HADES pairs a deterministic control MCU with a wireless / compute coprocessor, split by what each is good at.

```
                              HADES (the brain)
   +---------------------------------------------------------------+
   |                                                               |
   |   MSPM0G3507              UART              ESP32-S3 module    |
   |   (master)         <----------------->      (coprocessor)     |
   |   deterministic                              WiFi / BT /      |
   |   control          reset/boot GPIO --->      TinyML           |
   |                                                               |
   +------------------------------[ HADES bus ]--------------------+
                                       |
                                       |  signals + one clean supply
                                       |  (never raw battery / motor current)
                                       v
                              HADES shield (the muscle)
                          power stage / drivers / PDB
                                       |
                                       v
                              motors • servos • ESCs
```

| Role | Part | Handles |
|------|------|---------|
| **Master** | TI MSPM0G3507 (Cortex-M0+, 80 MHz) | Deterministic, timing-critical work: control loops, motor PWM, sensor timing, CAN-FD |
| **Coprocessor** | Espressif ESP32-S3 (module) | Non-deterministic work: WiFi/BT, telemetry, OTA, on-device TinyML |

**Why the MSPM0G3507 for control:** two advanced control timers (motor PWM), two simultaneous-sampling 12-bit 4 Msps ADCs plus on-chip op-amps and comparators (a real current-sense front-end), CAN-FD, and the MATHACL trig accelerator to soften the M0+'s lack of a hardware FPU.

**Why the ESP32-S3 for comms/ML:** native USB (clean flashing), WiFi/BT, and vector/SIMD instructions that ESP-NN uses to accelerate TensorFlow Lite Micro — so it does the wireless *and* the on-device ML without a third chip.

The two talk over UART, and the MSPM0 drives the ESP32's `EN`/`IO0` lines so the master can reset or bootload the coprocessor (recovery + firmware updates).

## The HADES bus (the shield contract)

The interface between the brain and its shields is a **frozen, versioned contract** — the most important thing in the whole project. Freeze it well and HADES can rev to v4 while old shields still fit, and a junior can build a brand-new shield for a motor class nobody imagined.

The contract has two halves:

- **Electrical** — a fixed pinout: PWM/servo channels, CAN-FD, I²C, UART, GPIO/ADC breakouts, **one clean regulated supply into the brain**, and reserved lines for **current/voltage-sense telemetry coming back** from the shield (INA-over-I²C style). _(Exact pin assignment: TBD.)_
- **Mechanical** — a fixed board outline, mounting-hole geometry, and stack orientation, so the whole shield family is physically interchangeable. _(Footprint: TBD.)_

**The power boundary is the rule that makes this safe:** across the bus, only signals and one clean supply cross. Raw battery voltage and motor current **never** touch the brain — they stay on the shield.

The contract is versioned independently of the boards (e.g. `HADES bus v1`). A HADES revision and a shield revision each declare which bus version they speak, so compatibility is always unambiguous.

## Planned features

_(Design targets — subject to change until v1 is frozen.)_

- Dual-MCU: MSPM0G3507 master + ESP32-S3 coprocessor over UART
- CAN-FD transceiver + connector (multi-board robots; industry-relevant)
- 3-pin servo/PWM header bank off the MSPM0 advanced timers
- Quadrature encoder inputs on timer channels
- Onboard 6-axis IMU on SPI _(part TBD)_
- Qwiic / STEMMA-QT I²C connector for off-the-shelf sensors
- USB-C on the ESP32-S3 with auto-boot/reset (no button dance to flash)
- SWD header for the MSPM0
- Input protection (reverse-polarity + TVS), per-rail power LEDs, a user LED per MCU
- Test points on every rail and the inter-MCU UART
- Clearly silkscreened, 5 V-tolerant pins marked

## Power

HADES only regulates its *own* clean rails — the muscle's power supply lives on the shield.

- A single step-down stage takes the input down to a 5 V rail; a downstream stage produces 3.3 V for the logic.
- USB-C and the shield-supplied input are OR'd through a priority mux so sources never fight.
- A low-noise LDO (e.g. LT3045-class) generates a separate quiet analog rail for the MSPM0's op-amp/ADC front-end.

The board's own electronics sip very little, so the input battery is sized by whatever the *shield* drives, not by the brain.

## Toolchain & programming

**MSPM0G3507**
- Code Composer Studio + the MSPM0 SDK / DriverLib (or Zephyr / FreeRTOS)
- Flash/debug over SWD with an **XDS110** (e.g. an MSPM0 LaunchPad's onboard probe) or a **CMSIS-DAP** probe via OpenOCD (TI ships `mspm0.cfg`)
- ⚠️ **ST-Link is not supported** — its firmware is locked to STMicroelectronics silicon and will refuse the TI target.

**ESP32-S3**
- ESP-IDF (or Arduino-ESP32)
- Native USB-C — no external programmer needed

## Repository structure

```
hades/
├── hardware/          # KiCad project (schematics, layout, gerbers)
│   ├── hades-brain/   # the main board
│   └── shields/       # HADES shield designs
├── firmware/
│   ├── mspm0/         # master firmware
│   └── esp32s3/       # coprocessor firmware
├── bus-spec/          # the HADES bus contract (pinout + mechanical), versioned
├── docs/              # getting-started, guides, board notes
└── README.md
```

## Getting started

> ⏳ **Coming with the first hardware revision.** Once boards exist, this section will walk you from unboxing to "both MCUs alive, LED blinking, WiFi up" in under an hour. Until then, the design lives under `hardware/` and the interface under `bus-spec/`.

## Roadmap

- [ ] Freeze **HADES bus v1** (pinout + mechanical footprint)
- [ ] Complete the brain schematic and layout (KiCad, 4-layer)
- [ ] Small validation batch (~5 boards) — hand to real users, watch what breaks
- [ ] Getting-started guide + docs
- [ ] First **HADES shield** (matched to the actual motors juniors use)
- [ ] Wider production run

## Contributing & succession

HADES is built to outlive any single maintainer. The project is designed to be handed down: the current maintainer documents it, then passes it to an interested junior with full guidelines, and the cycle continues.

If you're picking this up:
- The **bus spec** is sacred — don't change a frozen bus version; publish a new one.
- Keep the KiCad sources buildable and the docs current; a board nobody can re-order or re-open is a dead board.
- Prefer parts that will still be stockable in two years.

## License

_(To be finalized — recommended split:)_
- **Hardware:** CERN-OHL-S or TAPR OHL
- **Firmware:** MIT or Apache-2.0
- **Documentation:** CC BY 4.0

## Origin

Originally developed at **IIIT Naya Raipur** to give students a genuine step up from the Arduino Uno + ESP32 combo — a real embedded control platform for robotics, maintained and passed down from one cohort to the next.