<div align="center">

<img src="docs/media/logo.svg" width="96" alt="PocketBit pixel logo">

# PocketBit

**A DIY handheld game console you can build, hack and make games for.**

ESP32-P4 · two models: PocketBit and PocketBit Pro with an FPGA GPU · stereo sound · wireless multiplayer

![Status: designing](https://img.shields.io/badge/status-designing-f2b134.svg)
![ESP32-P4](https://img.shields.io/badge/CPU-ESP32--P4-E7352C?logo=espressif&logoColor=white)
![FPGA GPU](https://img.shields.io/badge/Pro-ECP5%20FPGA%20GPU-8a4fff.svg)
![Open hardware](https://img.shields.io/badge/open%20hardware-CERN--OHL--P-6abe30.svg)
[![Software: BitKit](https://img.shields.io/badge/software-BitKit-4aa3df.svg)](https://github.com/Dvrkstvr/bitkit)
[![Studio](https://img.shields.io/badge/try%20the%20studio-online-654FF0.svg)](https://dvrkstvr.github.io/bitkit/)

### [▶ Make games for it now](https://dvrkstvr.github.io/bitkit/) · [BitKit software](https://github.com/Dvrkstvr/bitkit)

<img src="docs/media/sprout-hop.gif" width="480" alt="Sprout Hop running on the BitKit runtime">

<sub>The same runtime you see here in the browser will run on the device.</sub>

</div>

---

## 🎯 What it is

PocketBit is the hardware half of the project: the PCB, the shell and the bill of materials.
The firmware, game runtime and browser studio live in **[BitKit](https://github.com/Dvrkstvr/bitkit)**.
You can already make and play games in the browser with a gamepad. When the board exists, the same carts will run on it.

## 🔧 Planned hardware

Two models, one software platform. Every BitKit game runs on both.

| | **PocketBit** | **PocketBit Pro** |
|---|---|---|
| Target price (parts, per device) | ~20 to 30 EUR | ~50 to 60 EUR |
| CPU | ESP32-P4 (dual-core RISC-V, 400 MHz, 32 MB PSRAM) | same |
| Radio | ESP32-C6 for ESP-NOW multiplayer, visits and trades | same |
| Graphics | P4 software renderer | **BitKit GPU** on a Lattice ECP5-25 FPGA + SDRAM: tiles, sprites, screen looks, 3D-lite |
| Screen | 2.8 to 3.2" 320×240 IPS | Bigger: 3.5 to 4", 480×320 or more |
| Emulators | GB / GBC / NES in software | Software emulators **plus** hardware cores for NES, SNES, GB/GBC and Master System |
| Controls | D-pad, A/B/X/Y, Start/Select, L/R shoulders | same, with Hall-effect analog triggers |
| Audio | Stereo speakers | same |
| Clock | RTC for idle/farming games that keep growing while it's off | same |
| Power | Li-ion with instant wake and pause/resume | same |
| Board | 4-layer | 6-layer (BGA FPGA) |

How it's built:

- **Chip-down custom PCB**, all SMD, fully assembled by JLCPCB or PCBWay. No dev boards or modules glued together.
- **Main board** with the chips, screen and power, plus **separate button boards** on the sides, connected with FFC cables.
- On the Pro, the P4 loads the FPGA at boot from its own flash, so the GPU updates with the firmware. Emulator cores are separate files on the SD card (`/cores`), loaded only when you start that emulator.
- No custom silicon. Prices are rough list-price estimates and will be firmed up with the real BOM.

The research behind the GPU and the two models: **[docs/research/fpga-gpu.md](docs/research/fpga-gpu.md)**.

## 🎮 Software highlights

- 60 fps locked target, with Lua games on a C runtime.
- A launcher home screen with utility apps (Home Assistant and more).
- GB / GBC / NES emulators on the Utilities row; hardware cores on the Pro.
- 2.5D on both models; low-poly 3D in hardware on the Pro.

## 🗺️ Roadmap

- [x] Runtime and browser studio ([BitKit](https://github.com/Dvrkstvr/bitkit))
- [x] BitKit device firmware and simulator (built for the ESP32-S3, moving to the P4)
- [x] Hardware decision: ESP32-P4 + ESP32-C6, two models
- [ ] Bench prototype: P4 dev board + display + buttons
- [ ] Pro bench prototype: P4 dev board + ECP5-25 board running the BitKit GPU
- [ ] Analog trigger and audio tests
- [ ] PocketBit schematic and PCB v1
- [ ] PocketBit Pro schematic and PCB v1 (reusing the PocketBit blocks)
- [ ] 3D-printable shell
- [ ] BOM and build guide

## 📁 Layout (as it fills in)

```
pcb/        KiCad schematic and board
shell/      3D-printable enclosure
bom/        Bill of materials
docs/       Build guide, photos, notes and research
```

## 🤝 Contributing

Ideas and hardware advice are very welcome. Open an issue, or try the [studio](https://dvrkstvr.github.io/bitkit/) and make something.

## 📄 Licence

Hardware designs will be released under **CERN-OHL-P v2** (permissive open hardware). BitKit software and the BitKit GPU gateware are MIT. Emulator cores for the Pro come from GPL projects, so they live in their own repository with full source and are loaded as separate files, never mixed into the MIT code.

<div align="center"><sub>Made for tinkerers who want to hold their games in their hands.</sub></div>
