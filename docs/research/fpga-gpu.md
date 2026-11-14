# BitKit GPU: an FPGA graphics chip next to the ESP32-P4

Research audit, 2026-11-14. Research only: no board design yet.

**Question:** PocketBit is moving to an ESP32-P4 (main CPU) + ESP32-C6 (radio, ESP-NOW). PCIe GPUs are not an option, so can we put a small FPGA next to the P4 as a graphics co-processor ("BitKit GPU": tiles, sprites, layers, maybe simple 3D)? Has anyone built this already?

## TL;DR

- **Nobody has published an ESP32-P4 + FPGA graphics build.** I searched GitHub, Hackaday, Hackster, CNX and the Espressif forum. The P4 handhelds that exist (for example the GB300-P4 transplant running Retro-Go) render in software. PocketBit would be the first public one.
- **The idea itself is proven.** "Small CPU + FPGA video chip" has shipped many times: Gameduino (SPI, 2011), Commander X16's VERA (iCE40UP5K), Xosera (iCE40UP5K), icestation-32, and two recent handhelds: **Game Bub** (ESP32-S3 + Artix-7, shipping 2026) and **SS Handheld** (STM32H723 + ECP5-85F with a 2D + 3D GPU, boards in fab).
- **The cheapest real option is a Lattice iCE40UP5K** (5.3k LUTs, 128 KB on-chip SRAM, fully open toolchain, QFN-48, roughly 6 to 13 USD). That is enough for a 320×240 tile + sprite PPU and screen effects, not for 3D. 3D needs an ECP5-25 class part (BGA-256, more layers, external RAM), which breaks the 40 to 50 EUR budget.
- **Honest caveat:** at 320×240 the P4 alone (400 MHz dual core, PPA blender, 2D-DMA) can probably push 2D at 60 fps. On the S3 the bottleneck measured so far is Lua, not pixel fill. An FPGA wins on full-screen looks/shaders, scanline effects, floor()/2.5D, bigger screens, and "zero CPU" sprites. It does not make Lua faster.
- **Recommendation:** prototype first, chip-down later. Wire a P4 dev board to a cheap iCE40 board over the P4's own i80 LCD bus, build a small MIT "BitKit PPU" (clean-room), and measure it against a P4-only renderer. If it wins, put an iCE40UP5K on the main board between the P4 and the screen, configured by the P4 at boot (no flash chip). Keep 3D as a later ECP5 option.

## 1. Has anything like this been built?

### ESP32-P4 + FPGA

Nothing found. Closest P4 work:

| Project | What it is | FPGA? |
|---|---|---|
| [GB300-P4](https://hackaday.com/2026/01/14/esp32-p4-powers-retro-handheld-after-a-transplant/) | Commercial GB300 handheld with its chip swapped for a P4, runs Retro-Go (incl. SNES/Genesis) and DOOM | No, software rendering |
| [esp32-p4-library / esp32_3d_engine](https://github.com/andresragot/esp32-p4-library) | Software 3D engine for the P4 | No |
| [4D Systems GFX4dESP32P4](https://github.com/4dsystems/GFX4dESP32P4) | Graphics library for P4 display modules | No |

### Other ESP32 + FPGA boards

| Project | CPU + FPGA | Role of the FPGA | Licence | Notes |
|---|---|---|---|---|
| [Game Bub](https://www.cnx-software.com/2025/08/27/fpga-based-game-bub-handheld-console-supports-original-nintendo-cartridge-wireless-controllers/) | ESP32-S3 + Artix-7 XC7A100T | Whole GB/GBC/GBA emulation; ESP32 does Wi-Fi and system | GPLv3 + CC BY-SA | Closest "ESP32 + FPGA handheld". 4" 720×480, 249 USD. FPGA is the console, not a GPU for the ESP32 |
| [ULX3S](https://radiona.org/ulx3s/) | ECP5 + ESP32 | FPGA does everything; ESP32 is Wi-Fi/loader helper | Open hardware | Big ecosystem of video cores, HDMI out |
| [ICE-V Wireless](https://github.com/ICE-V-Wireless/ICE-V-Wireless) | ESP32-C3 + iCE40UP5K | General purpose; **ESP32 loads the FPGA bitstream over SPI** | Open | Proves the "MCU configures iCE40, no flash" trick we want |
| [Spartan Edge Accelerator graphics](https://hackaday.io/project/175763-spartan-edge-accelerator-graphics-esp32fpga) | ESP32 + Spartan-7 XC7S15 | Tile/sprite PPU, 720p HDMI | Not stated | 2020 hobby experiment, inactive. Same idea as ours, on HDMI |
| [pico-ice](https://cnx-software.com/?p=114836) | RP2040 + iCE40UP5K | General purpose | Open | Pattern for an MCU + iCE40 on one small board |

### MCU + FPGA graphics chips (not ESP32)

| Project | FPGA | Host link | What it draws | Licence |
|---|---|---|---|---|
| [SS Handheld Console](https://github.com/badaboombox/SS-Handheld-Console) | ECP5-85F (BGA-381) + SDRAM | STM32H723 over a parallel bus | Tile/sprite/affine 2D + textured 3D (RasterIX based), 480×272 RGB panel, 60 fps | RTL GPL-3.0, hardware CERN-OHL-P |
| [Gameduino 1](https://github.com/Godzil/gameduino) | Spartan-3A | **SPI**, looks like 32 KB of RAM to the CPU | 400×300, 96 sprites per line, tiles, collision | Code GPL, hardware BSD |
| Gameduino 3X Dazzler | BT815 + Spartan-7 | SPI | HDMI 720p, BT815 display lists | Mixed |
| Commander X16 VERA | iCE40UP5K | 8-bit parallel | 640×480, 2 layers, 128 sprites | Check before reuse |
| [Xosera](https://github.com/XarkLabs/Xosera/) | iCE40UP5K | 8-bit parallel, 16 registers | Bitmap/tile/text modes, blitter, "copper" | Check LICENSE before reuse |
| [icestation-32](https://github.com/dan-rodrigues/icestation-32) | iCE40UP5K / ECP5 | Own RISC-V CPU inside | SNES-style VDP: layers, sprites, affine | Check before reuse |
| Pimoroni PicoVision | 2× RP2040 (no FPGA) | QSPI PSRAM shared | Second RP2040 acts as the GPU | Example of the "GPU chip" idea without an FPGA |
| Bitluni 8192-core GPU | Thousands of tiny RISC-V MCUs | n/a | Stunt build | Fun, not practical |

**Takeaway:** a 320×240 tile/sprite PPU on an iCE40UP5K is well-trodden (VERA and Xosera do more on the same chip). A textured 3D rasterizer needs an ECP5-25 or bigger (RasterIX uses ~10.7k LUTs + 19 DSPs on a Spartan-7 25 plus external RAM; SS Handheld uses an ECP5-85).

## 2. Open GPU / sprite cores and their licences

PocketBit stays MIT (software/gateware) + CERN-OHL-P (hardware). GPL cores are **ideas only**, same rule as Lilka.

| Core | Licence | Use it how |
|---|---|---|
| [rj45/vdp](https://github.com/rj45/vdp) | **MIT** | Can borrow code. ECP5/ULX3S, tiles + sprite sheets done, sprites not finished |
| [Yosys](https://github.com/YosysHQ/yosys), nextpnr | ISC | Toolchain, fine |
| [Apicula](https://github.com/yosysHQ/apicula) (Gowin) | MIT | Toolchain, fine |
| [RasterIX](https://github.com/ToNi3141/Rasterix) | GPL-3.0+ | Ideas only (3D pipeline, OpenGL ES 1.1 subset, has a Pico host) |
| SS Handheld RTL | GPL-3.0+ | Ideas only |
| Game Bub gateware | GPLv3 | Ideas only |
| Gameduino 1 Verilog | GPL | Ideas only (its "GPU looks like RAM over SPI" model is a great reference) |
| Xosera, icestation-32, VERA, [rastrgrafx](https://github.com/toivoh/rastrgrafx) | Not confirmed | Read their LICENSE before copying anything; until then ideas only |

Bitstreams made by open toolchains carry no licence from the tool, so a self-written MIT PPU stays MIT.

## 3. How the FPGA would sit next to the P4

### Option A (recommended): FPGA owns the screen, P4 sends commands

```
 ESP32-C6 ──SDIO/SPI── ESP32-P4 ──16-bit i80 bus── FPGA "BitKit PPU" ──i80 or RGB── LCD
 (radio)               (Lua, game,   (P4 thinks the       (VRAM, tiles, sprites,
                        audio, 2D-DMA) FPGA is an LCD)      palette, looks, scanout)
```

- The P4 already has an i80 LCD driver (`esp_lcd` i80 bus, DMA). The FPGA pretends to be an LCD controller with extra registers: write a register address as a "command", then stream data. No new P4 driver work beyond a thin `bk_gpu_*` layer.
- 16 bits at 20 to 40 MHz gives 40 to 80 MB/s. A whole 320×240×16-bit frame is 154 KB, 9.2 MB/s at 60 fps, so even "P4 draws everything and FPGA only does looks" fits easily. Tile/sprite updates per frame are a few KB.
- The FPGA renders scanlines on the fly (racing the beam, like VERA/Gameduino) into a line buffer and pushes them to the panel. On an ST7789 over i80 the FPGA is the bus master; on an RGB panel it generates the timing itself.
- One interrupt line (vblank/TE) back to the P4 keeps the 60 Hz tick locked.

### Option B: QSPI/octal SPI link

The P4's SPI2 supports 1/2/4/8 lines. Fewer pins (6 to 11) than i80, works fine for commands and asset uploads, a bit slower for full frames. Gameduino proves the model. Good fallback if pins get tight.

### Option C: FPGA as a render accelerator with readback

The P4 keeps the framebuffer, the FPGA renders into it and sends results back over PARLIO RX. Two-way traffic, more latency, more complex. Not worth it.

### Option D: MIPI-DSI pass-through

P4 DSI (2 lanes) into an FPGA with a hard MIPI D-PHY (Gowin GW5A-25 has one, iCE40 does not), FPGA overlays sprites and re-drives a DSI panel. Only makes sense with a big MIPI screen. Overkill for 320×240.

## 4. What the BitKit PPU would do (iCE40UP5K sized)

Sized for 320×240 and the 128 KB SPRAM + 15 KB BRAM inside the iCE40UP5K:

- 2 tile layers (8×8, 4 or 8 bpp), smooth scroll, wrap, per-line scroll registers (raster()/scanline effects in hardware)
- 64 to 128 sprites, up to ~32 per line, flip, priority, palette per sprite
- 256-colour palette to RGB565, fade/shade tables (matches shade() and display() looks)
- Screen looks in the output stage: scanlines, CRT tint, dither, brightness
- Optional: a Mode-7 style affine floor layer for floor()/Kart Lab (needs the DSP multipliers; fits if sprites are trimmed)
- Not in scope on iCE40: textured 3D. That is the ECP5 "Phase 8" option.

Memory budget: 320×240 at 8 bpp is 75 KB, so even a full indexed framebuffer fits in the 128 KB SPRAM, which also allows a "bitmap layer" fallback for games that draw with pset()/line().

## 5. FPGA candidates

Prices are from distributor pages on 2026-11-14; LCSC live stock could not be fully checked from here, so verify before ordering.

| Part | LUTs | On-chip RAM | Package | Open toolchain | Price seen | JLC/LCSC | Fit |
|---|---|---|---|---|---|---|---|
| **Lattice iCE40UP5K-SG48I** | 5,280 | 128 KB SPRAM + 15 KB BRAM, 8 DSP | QFN-48 (7×7) | Yosys + nextpnr-ice40, mature | ~6 to 8 USD at Western distributors; LCSC lists the WLCSP version from 12.63 USD (out of stock) | [C2678152](https://jlcpcb.com/partdetail/Lattice-ICE40UP5KSG48I/C2678152), extended part | **Best for 2D PPU** |
| Gowin GW1NR-LV9QN88 | 8,640 | 64 Mbit PSRAM in package | QFN-88 | Apicula (supported) | Mouser 27 to 37 USD; much cheaper in Chinese channels (Tang Nano 9K board ~15 to 20 USD) | Check LCSC | Good, but price via Western/LCSC kills the budget |
| Gowin GW2AR-LV18QN88 | 20,736 | 64 Mbit SDRAM in package | QFN-88 | Apicula (supported) | [LCSC 41.33 USD, 89 in stock](https://www.lcsc.com/product-detail/C380550.html) | C380550 | Could do 3D-lite, too expensive |
| Gowin GW5A-LV25 | 23,040 | 1 Mbit BSRAM, 28 multipliers, hard MIPI | BGA-121 | Apicula GW5A in progress (NLnet funded) | Board-level only seen | Check | Only if MIPI pass-through is wanted |
| Lattice LFE5U-25F-6BG256C | 24,000 | 1 Mbit BRAM, 28 DSP | BGA-256 (0.8 mm) | Yosys + nextpnr-ecp5 (Trellis), mature | [LCSC from 11.51 USD (out of stock)](https://lcsc.com/product-detail/Programmable-Logic-Device-CPLDs-FPGAs_Lattice-LFE5U-25F-6BG256C_C1521614.html) | C1521614 | **Best for 3D later**, needs 6-layer board + RAM |

Both Lattice parts are the safe pick for an open, community-friendly project: the open toolchains are mature and nobody needs a vendor licence to build the gateware.

## 6. Power

Rough numbers, to confirm on hardware:

- iCE40UP5K: a few mW static, tens of mW running a PPU at ~25 to 50 MHz. Negligible next to the P4 and backlight.
- ECP5-25: around 100 to 300 mW depending on clock and use, plus external RAM. That is a noticeable cut in battery life.
- Rails: iCE40 needs 1.2 V core + 3.3 V I/O (+2.5 V only if using its own SPI flash path; the P4-configured mode avoids most of that). A small LDO or the PMIC's spare buck is enough.

## 7. Cost impact against the 40 to 50 EUR budget

| Add-on | iCE40UP5K path | ECP5-25 path |
|---|---|---|
| FPGA | ~6 to 13 EUR | ~11 to 15 EUR (when in stock) |
| Config flash | 0 (P4 loads the bitstream over SPI at boot, like ICE-V Wireless) | 0 to 0.5 EUR |
| External RAM | 0 (SPRAM is enough) | 2 to 4 EUR (HyperRAM/SDRAM) |
| Regulators, oscillator, passives | ~0.5 to 1 EUR | ~1 to 2 EUR |
| PCB | Same 4-layer board, QFN is easy for JLC | BGA-256 pushes to 6 layers, +3 to 6 EUR at small runs |
| JLC extended part fees | ~3 USD per extended part per order (not per board) | same |
| **Per device** | **~+7 to 15 EUR** | **~+17 to 27 EUR** |

The iCE40 path is tight but possible inside 50 EUR if the P4 module and screen stay as planned. The ECP5 path does not fit the budget on the main board.

## 8. Recommendation

1. **Build it, but as a measured experiment first.** Nothing like it exists for the P4, the concept is proven elsewhere, and the iCE40UP5K is cheap and fully open. That is a good "only PocketBit does this" feature for an open, hackable console.
2. **Phase G1, bench prototype (about 30 to 60 EUR once):** ESP32-P4 dev board + iCEBreaker/UPduino (iCE40UP5K) + the ST7789 screen, wired over 16-bit i80. Write a clean-room MIT "BitKit PPU" in Verilog (or Amaranth), with a cycle-accurate model in the BitKit repo so the browser studio and `bitsim` render exactly what the PPU renders.
3. **Measure against P4-only.** Same demo carts (Kart Lab, Deepglow looks, Sweep Storm) on the P4 with PPA/2D-DMA vs with the PPU. Go chip-down only if the PPU frees real CPU time for Lua or makes the looks run at 60 fps where software can't.
4. **Phase G2, chip-down:** iCE40UP5K-SG48 on the main board between P4 and screen, bitstream stored in the P4 flash and loaded at boot (updatable with the firmware). Keep a solder-jumper/0 Ω option so the P4 can drive the screen directly, so a board with a bad FPGA or an unfinished PPU still boots.
5. **Phase G3 (Phase 8, 3D):** a separate ECP5-25 "GPU module" board on a board-to-board connector, for people who want 3D. Not on the 50 EUR base model.
6. **Licences:** write the PPU from scratch under MIT. Use GPL projects (RasterIX, SS Handheld, Game Bub, Gameduino) only as ideas and credit them in "Credits and inspiration". rj45/vdp (MIT) is the one core we could legally borrow from.

## 9. Rough architecture

```
                 ┌──────────────────────────── main board ─────────────────────────────┐
  battery ─ PMIC ┤  3.3 V / 1.8 V / 1.2 V                                               │
                 │                                                                      │
  ESP32-C6 ◄─SPI/SDIO─► ESP32-P4 (+32 MB PSRAM, 16 MB flash)                            │
  (ESP-NOW)      │       │  Lua runtime, game logic, audio (I2S), input (ADC/SPI),       │
                 │       │  holds PPU bitstream in flash, loads it at boot via SPI       │
                 │       │                                                              │
                 │       ├─ 16-bit i80 (D0-15, WR, RD, DC, CS) ──►  iCE40UP5K PPU      │
                 │       ◄─ VBLANK IRQ, CDONE ─────────────────────  128 KB SPRAM VRAM │
                 │       ├─ SPI (CRESET, SS, SCK, MOSI) for config ─►  tiles/sprites/   │
                 │                                                    palette/looks     │
                 │                                                       │              │
                 │                                       8/16-bit i80 or RGB            │
                 │                                                       ▼              │
                 │                                             LCD (ST7789 320×240,     │
                 │                                              or bigger later)        │
                 └──────────────────────────────── FFC ─► button PCBs ─────────────────┘
```

Software side (BitKit): a `bk_gpu` layer with two backends, "software" (today's renderer, used in the browser and on boards without a PPU) and "ppu" (sends tile maps, sprite tables, palette and look registers each vblank). Lua APIs (spr, map, pal, shade, display, raster, floor) stay the same, so carts run on both.

## Open questions for Calvin

- Is a bigger screen (480×320 or 640×480) on the table? That is where the PPU pays off most.
- Is 3D a must on the base model, or fine as an optional GPU module?
- Verilog or Amaranth (Python) for the PPU? Amaranth may suit the community better; Verilog has more examples.

## Sources

- GB300-P4: https://hackaday.com/2026/01/14/esp32-p4-powers-retro-handheld-after-a-transplant/
- Game Bub: https://www.cnx-software.com/2025/08/27/fpga-based-game-bub-handheld-console-supports-original-nintendo-cartridge-wireless-controllers/
- SS Handheld Console: https://github.com/badaboombox/SS-Handheld-Console
- Spartan Edge Accelerator graphics: https://hackaday.io/project/175763-spartan-edge-accelerator-graphics-esp32fpga
- ICE-V Wireless: https://github.com/ICE-V-Wireless/ICE-V-Wireless
- ULX3S: https://radiona.org/ulx3s/
- pico-ice: https://cnx-software.com/?p=114836
- Gameduino 1 source: https://github.com/Godzil/gameduino
- Xosera: https://github.com/XarkLabs/Xosera/
- icestation-32: https://github.com/dan-rodrigues/icestation-32
- rj45/vdp: https://github.com/rj45/vdp
- rastrgrafx: https://github.com/toivoh/rastrgrafx
- RasterIX: https://github.com/ToNi3141/Rasterix
- Apicula: https://github.com/yosysHQ/apicula
- Bitluni GPU: https://hackaday.com/2026/07/06/gluing-8192-mcus-together-to-make-a-gpu/
- PicoVision: https://www.tomshardware.com/reviews/pimoroni-picovision
- ESP32-P4 datasheet: https://documentation.espressif.com/esp32-p4_datasheet_en.html
- ESP32-P4 PPA: https://docs.espressif.com/projects/esp-idf/en/v5.5/esp32p4/api-reference/peripherals/ppa.html
- ESP32-P4 PARLIO: https://docs.espressif.com/projects/esp-idf/en/v5.4/esp32p4/api-reference/peripherals/parlio.html
- Tang Primer 25K (GW5A): https://wiki.sipeed.com/hardware/en/tang/tang-primer-25k/primer-25k.html
- LCSC iCE40UP5K: https://www.lcsc.com/product-detail/C1519053.html, JLC C2678152: https://jlcpcb.com/partdetail/Lattice-ICE40UP5KSG48I/C2678152
- LCSC GW2AR-18: https://www.lcsc.com/product-detail/C380550.html
- Mouser GW1NR-9: https://www.mouser.com/en/ProductDetail/GOWIN-Semiconductor/GW1NR-LV9QN88PC6-I5
- LCSC LFE5U-25F: https://lcsc.com/product-detail/Programmable-Logic-Device-CPLDs-FPGAs_Lattice-LFE5U-25F-6BG256C_C1521614.html
