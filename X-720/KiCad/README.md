# X-720 recreation: KiCad project

Open `X-720.kicad_pro` in KiCad.

- The files are in **KiCad 7** format, so KiCad 7, 8 and 9 can all open them. Newer versions upgrade the files when you save.
- `X-720_schematic.pdf` is a plot of the current sheets, for viewing without KiCad.

## Sheets

| Sheet | File | Status |
|---|---|---|
| Root | `X-720.kicad_sch` | Holds the sheet symbols and the plan for the other sheets |
| Bus Interface | `bus_interface.kicad_sch` | **v0.1 drawn**: decode logic simulated, netlist verified |
| Video Core | | To do: MC6847, 3 x 2 KB VRAM, CG ROM, +1K page logic, address mux |
| TV BASIC ROM | | To do: A000-AFFF, socketed EEPROM |
| Video Output | | To do: RGB / sync / sound to SCART |
| Power | | To do: NiCd + adaptor, 5 V regulator, VCC2 level detect |

Nets between sheets use **global labels**. Nets used only on one sheet are local labels.

## Bus Interface (v0.1)

| Ref | Part | Function |
|---|---|---|
| J1 | 40-pin | X-07 rear extension connector. Pinout from the X-07 User's Guide. |
| J2 | 40-pin | Pass-through slot for I/O packs, wired 1:1 to J1 |
| U1 | 74HCT373 | Latches A0-A7 from the multiplexed AD0-AD7 bus on ALE (→ `LA0`-`LA7`) |
| U2 | 74HCT244 | Buffers A8-A15 (→ `B_A8`-`B_A15`) |
| U3 | 74HCT244 | Buffers /RD, /WR, IO/M, ALE, /RESET |
| U4 | 74HCT245 | Data transceiver, X-07 bus ↔ internal `D0`-`D7`. DIR = /RD. Enabled only by `~X720_SEL`. |
| U5 | 74HCT138 | Memory decode in 2 KB blocks: 8000, 8800, 9000 (VRAM), 9800 (slot ROM), A000+A800 (`~ROM_CS`) |
| U6 | 74HCT138 | I/O decode: 80-87, 88-8F (CG ROM), 90-97 (mode/status), 98-9F (slot latch) |
| U7, U8, U9, U10 | 74HCT04 / 08 / 21 / 32 | Glue logic: IOREQ, ROM select, X-720 select, read/write strobes |
| U11 | 74HCT374 | Port 90 write: mode register (CSS0, CSS1, GM0-GM2, +1K, A/G, FSR) |
| U12 | 74HCT244 | Port 90 read: status (D7 = FS, D6 = INT flag, D5-D0 = 0) |
| C1-C13 | | 100 nF per IC, plus 10 µF bulk |

### Decisions

- **74HCT instead of 74LS.** Much lower current for a battery-powered unit, and inputs still accept TTL levels.
- **Same structure as the original.** The bus logic follows figure 11, not a GAL/CPLD, so it can be checked against the scan.
- **U4 is gated.** It only drives the X-07 bus for 8000-97FF, A000-AFFF and ports 80-97. 9800-9FFF and port 98-9F belong to the pass-through slot.
- **Spare gate inputs tied to GND.** Unused outputs are marked no-connect.
- **Footprints.** DIP through-hole for the logic. J1/J2 have no footprint yet: the connector pitch and part are unknown.

### Verification (18 Nov 2026)

Done with KiCad 7.0.11 `kicad-cli`. The KiCad 7 command line has no ERC, so the netlist was checked by script.

- The exported netlist was compared against the intended pin-to-net list (295 connections):
  - No missing, split or merged nets.
  - The 17 no-connect pins are all deliberate.
- The decode logic was simulated from the exported netlist over every memory address (IO/M = 0) and every port (IO/M = 1). The selects match the X-07 memory and I/O map exactly.

### Expected ERC findings until the other sheets exist

When you run ERC in KiCad, the following will be flagged. All of them are intentional for now:

- **Global labels used only once.** These are the signals for sheets not yet drawn: `LA0`-`LA2`, `B_A8`-`B_A10`, `B_~RESET`, the mode bits, `~MEM_9800`, `~SLOT_WR`.
- **Undriven inputs** on `FS` and `INT_FLAG`. These are driven by the Video Core sheet.
- **`+5V` and `GND`.** Both carry PWR_FLAGs until the Power sheet provides the regulator.

### Open items for this sheet

- J1/J2 connector part and footprint.
- The original's series parts on the X-07 side, not yet reproduced:
  - 1 Ω (R351/R352) in the GND pins.
  - 120 Ω (R250) on VCC2.
  - Switch and 4.7 nF on KACL.
- Port 98-9F slot address latch (`~SLOT_WR`). Its function is not yet understood.
- `X_INTR`: the cursor-blink interrupt from FS/FSR, to be completed on the Video Core sheet.
- Mode register reset: the 374 has none, so the TV BASIC ROM must initialise port 90.
