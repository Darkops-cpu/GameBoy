# HADES


**A dual-MCU control brain for people who've outgrown the Arduino Uno + jumper-wire ESP32 combo.**

HADES is a control board built around one idea: it is the *brain*, not the *muscle*. It runs deterministic control loops and talks to the network — it does **not** carry motor current. Power for actuators lives on a separate, swappable **HADES shield**, so the brain stays clean, cool, and reusable across every robot it gets bolted to.

This repo is also the successor to an earlier, already-taped-out board (internally called **GameBoy**, see [`OLD Board/`](./OLD%20Board)) that proved the dual-MCU concept and is being folded into the brain/shield architecture described here.

> **Status:** 🚧 In active design. `New Board/Hades` is a fresh KiCad 9 project — USB-C input, ESD protection, and power-path parts are placed; the MSPM0/ESP32 core, the bus connector, and the rest of the schematic are not yet drawn. Nothing in this revision is manufactured. Everything marked _(TBD)_ below is not yet final. The **OLD Board** is a finished, previously-fabricated design (schematic + PCB renders present) and is the best current reference for "does this dual-MCU idea actually work."

---

## Table of contents

- [Philosophy: brain, not muscle](#philosophy-brain-not-muscle)
- [Architecture](#architecture)
- [The HADES bus (the shield contract)](#the-hades-bus-the-shield-contract)
- [Planned features](#planned-features)
- [Power](#power)
- [Toolchain & programming](#toolchain--programming)
- [Repository structure](#repository-structure)
- [Current status](#current-status)
- [Lineage: OLD Board (GameBoy) → HADES](#lineage-old-board-gameboy--hades)
- [Getting started](#getting-started)
- [Roadmap](#roadmap)
- [Why this project exists](#why-this-project-exists)
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

This is what's actually in the repo today — not the target layout yet (see [Roadmap](#roadmap) for the reorganization into `hardware/` / `firmware/` / `bus-spec/` once the brain/shield split is real):

```
HADES/
├── New Board/
│   └── Hades/                    # current KiCad 9 project — the HADES "brain" board
│       ├── Hades.kicad_pro
│       ├── Hades.kicad_sch       # schematic (in progress — power in, MCUs not yet placed)
│       ├── Hades.kicad_pcb       # 2-layer layout, early
│       ├── Hades-backups/        # KiCad auto-backups (timestamped .zip snapshots)
│       └── README.md
├── OLD Board/                    # predecessor design, "GameBoy" — finished, previously fabricated
│   ├── README.md                 # full spec sheet + design-decision rationale
│   ├── Schematic/                # V1.0 – V1.3 schematic PDFs
│   ├── Docs/                     # V1.0 – V1.3 per-revision changelogs
│   ├── PCB 3D/                   # rendered board images (front/back, V1 & V2)
│   └── Reference/                # ESP32/MSPM0 datasheets + design references
└── README.md                     # this file
```

## Current status

| Track | State |
|---|---|
| **OLD Board (GameBoy)** — architecture proof | ✅ Schematic finalized through V1.3, PCB rendered, previously fabricated |
| **New Board (HADES brain)** — schematic | 🔄 Power input (USB-C, ESD protection) placed; MSPM0G3507 + ESP32-S3 core not yet drawn |
| **New Board (HADES brain)** — layout | 🔄 2-layer board started; routing pending schematic completion |
| **HADES bus (shield contract)** | ⏳ Not yet frozen — pinout and mechanical footprint TBD |
| **HADES shield** | ⏳ Not started |
| **Firmware (either MCU)** | ⏳ Not started — waiting on hardware |

## Lineage: OLD Board (GameBoy) → HADES

The `OLD Board/` design (codename **GameBoy**, no relation to Nintendo) is where the dual-MCU idea — TI **MSPM0G3507** as the deterministic master, **ESP32-S3-WROOM-1** as the wireless/compute coprocessor over UART — was designed, schematic-captured, and taken through four documented revisions:

| Rev | Change | Why |
|---|---|---|
| V1.0 | Single MCU (MSPM0G3507) + CH340E debug UART + LIS2DH accelerometer | Baseline: get the master MCU alive and talking |
| V1.1 | Added ESP32-S3-WROOM-1 coprocessor, CP2102N + auto-boot circuit, USB-C | Wireless/compute half of the architecture, flashable without a button dance |
| V1.2 | Added TP4056 Li-ion charging + BJT/MOSFET power-path switching | Untethered, battery-capable operation |
| V1.3 | **Removed** the V1.2 battery circuitry | Simplify the power path for the current build phase — a real "found the hard way, fixed" cycle, not just additive changes |

That revision history is the same succession pattern this repo is designed to keep running: something breaks or turns out unnecessary, the next revision documents why and fixes it, and the fix is traceable in `OLD Board/Docs/`.

`New Board/Hades` is the next link in that chain — it keeps the dual-MCU pairing but re-architects around the **brain/shield split** described above, instead of putting motor drive and battery management on the same board as the control MCU.

## Getting started

> ⏳ **Coming with the first hardware revision.** Once brain boards exist, this section will walk you from unboxing to "both MCUs alive, LED blinking, WiFi up" in under an hour, with no hand-wired jumper diagram involved. Until then:
> - To see the dual-MCU architecture working end-to-end today, read [`OLD Board/README.md`](./OLD%20Board/README.md) and the V1.0–V1.3 docs in [`OLD Board/Docs/`](./OLD%20Board/Docs).
> - To follow or contribute to the current board, open [`New Board/Hades/Hades.kicad_sch`](./New%20Board/Hades/Hades.kicad_sch) in KiCad 9.

## Roadmap

- [ ] Complete the HADES brain schematic (MSPM0G3507 + ESP32-S3 core, current-sense front-end, CAN-FD transceiver)
- [ ] Freeze **HADES bus v1** (pinout + mechanical footprint)
- [ ] Finish the 4-layer brain PCB layout
- [ ] Small validation batch (~5 boards) — hand to real users, watch what breaks
- [ ] Reorganize the repo into `hardware/` / `firmware/` / `bus-spec/` / `docs/` once the brain/shield split is fabricated
- [ ] Getting-started guide + docs (the "forty minutes to a sweeping servo" experience)
- [ ] First **HADES shield** (matched to the actual motors used on club robots)
- [ ] Wider production run + institutional handoff to a junior maintainer

## Why this project exists

The measure of success here isn't the PCB — it's what happens after it. HADES is meant to stop being "someone's project" and become "the board": something a first-year reaches for in week three without a wiring diagram, something the club's competition robots run on under real deadline pressure, something that keeps getting revised by people who never met the original designer.

Concretely, that means:
- **Faster than the breadboard, not just different from it.** If reaching for a HADES isn't the obviously easier choice next to a fistful of dupont wires, the board hasn't earned its place yet.
- **Survives contact with students and with robots.** Bench demos are cheap; a board holding a current-sense loop through a stalled motor for real, under competition deadline, is the actual test.
- **The revision chain has to turn without the original designer.** Rev 1 will have a quirk someone finds the hard way (the OLD Board's V1.0→V1.3 history is exactly that pattern already). What matters is whether rev 2 gets fixed and shipped by someone else, using docs good enough that they didn't need to ask.

Almost every student hardware project dies on two things: **documentation nobody wrote** and **a handoff that never happened**. Everything in [Contributing & succession](#contributing--succession) below exists to clear those two bars on purpose, rather than assuming they'll clear themselves.

## Contributing & succession

HADES is built to outlive any single maintainer. The project is designed to be handed down: the current maintainer documents it, then passes it to an interested junior with full context, and the cycle continues — the same way `OLD Board/Docs/V1.0.md` through `V1.3.md` already document what changed and why at each step.

If you're picking this up:
- The **bus spec** is sacred — don't change a frozen bus version; publish a new one.
- Keep the KiCad sources buildable and the docs current; a board nobody can re-order or re-open is a dead board.
- Document *why*, not just *what* — follow the pattern in `OLD Board/Docs/`: what changed, and the reason, per revision.
- Prefer parts that will still be stockable in two years.
- If you're the one stepping back, name your successor before you go quiet — a maintainer chain with a gap in it is how projects die.

## License

_(To be finalized — recommended split:)_
- **Hardware:** CERN-OHL-S or TAPR OHL
- **Firmware:** MIT or Apache-2.0
- **Documentation:** CC BY 4.0

## Origin

Originally developed at **IIIT Naya Raipur** to give students a genuine step up from the Arduino Uno + ESP32 combo — a real embedded control platform for robotics, maintained and passed down from one cohort to the next.
