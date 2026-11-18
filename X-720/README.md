# Canon X-720 colour video interface: research notes

Last updated 18 Nov 2026, after reading the X-720 manual (FR) and the book "Applications en assembleur dans l'univers du Canon X-07" (ch. 6, pp. 69-78).

Extracted figures from the book are in [`Book_extracts/`](Book_extracts/):

- [Figure 8: block diagram](Book_extracts/fig08_block_diagram.png)
- [Figure 9: I/O port table](Book_extracts/fig09_io_port_table.png)
- [Figure 10: VRAM usage per mode](Book_extracts/fig10_vram_map.png)
- Figure 11: full schematic, as a [full spread](Book_extracts/fig11_schematic_full_spread.png), [left page (p. 75)](Book_extracts/fig11_schematic_left_page75.png), and [right page (p. 76)](Book_extracts/fig11_schematic_right_page76.png)
- Text pages: [hardware intro](Book_extracts/pages69-70_hardware_intro.png), [I/O ports](Book_extracts/pages71-72_io_ports.png), [ROM entry points](Book_extracts/pages77-78_rom_entry_points.png)

## What the X-720 is

- Colour TV/monitor interface for the Canon X-07 (1983).
- Plugs into the X-07's 40-pin rear extension bus.
- Has its own slot for plug-in extension modules (I/O packs), with the bus passed through.

French manual spec sheet (p. 11):

| Item | Value |
|---|---|
| Output | RGB + sync + sound on a male SCART (Péritel) plug. For a SECAM TV without SCART, use an antenna adaptor (e.g. CGV PHS 60). |
| Power | 6 V DC NiCd battery, plus the AD-5 mains adaptor (pocket.free.fr says "8 V adaptor"; check against the AD-5's label) |
| Power draw | 1.4 W |
| Operating temperature | 0-40 °C |
| Size | 220 x 270 x 53.5 mm |
| Weight | 1.3 kg |

The book's schematic is a different variant: NTSC composite plus RF (3.579545 MHz crystal, RF modulator, ch1/ch2 switch). The core logic is documented, but the French RGB output stage is **not**.

## Screen modes (manual p. 16; set with `SCREEN`)

1. Text, 32 x 16
2. Semi-graphics, 64 x 48
3. Colour standard definition, 128 x 96, 4 colours
4. Mono standard definition, 128 x 192
5. Colour high definition, 128 x 192, 4 colours
6. Mono high definition, 256 x 192

## Confirmed: Motorola MC6847 design (book p. 69, figure 8)

- **Video RAM:** 6 KB as three 2 KB SRAMs. One of them holds the attribute (colour) signal.
- **ROM:** two packages:
  - CG ROM (displayable characters)
  - TV BASIC ROM (BASIC extension)

### Parts read off figure 11

The scan is about 200 dpi, so verify every marking before relying on it.

| Ref | Part | Role |
|---|---|---|
| (unnumbered) | MC6847P | Video display generator |
| IC203, IC204, IC205 | 6117 (2K x 8 SRAM) | VRAM 8000-97FF; IC205 holds the attribute bytes |
| IC220 | Toshiba TC533xP mask ROM | TV BASIC ROM, A000-AFFF |
| IC104 | Toshiba TC533xP mask ROM | CG ROM, read by the CPU through I/O 80-8F |
| IC213 | 74LS373 | Latches A0-A7 from the multiplexed AD bus on ALE |
| IC206 | 74LS138 | Address/IO decode |
| IC202 | 74LS139 | Address/IO decode |
| IC214, IC219, IC223, IC107, IC110 | 74LS244 | Buffers |
| IC208 | 74LS365 | Buffer |
| IC221, IC210 | 74LS245 | Data transceivers between CPU bus and VRAM bus |
| IC111, IC114 | 74LS374 | Mode/control registers and latches |
| IC101 | 74LS157 | Address mux (CPU vs 6847 access to VRAM/CG) |
| (unnumbered) | 74LS193 (probably) | Counter near FS (row counter for CG ROM addressing) |
| 201, 209, 211, 215, 216, 217, 222, 227, 113 | 74LS00/02/04/32/86-type gates | Glue logic |
| (unnumbered) | MC1372-like (marking unclear) | Colour/RF encoder (NTSC version) |

Other parts of the circuit:

- **Video output (NTSC version):** the colour encoder runs from a 3.579545 MHz crystal. It feeds video amp transistors TR4-TR6, a VIDEO OUT jack, and an RF modulator.
- **Power supply:**
  - 5-cell NiCd battery, charged through D1-D3 and R1 (27 Ω).
  - Discrete regulator built from TR1-TR3, TR7, TR8, ZD1 and VR1.
  - A level comparator on the X-07's VCC2 shuts the X-720 supply down when the X-07 is off.
- **Boards:** a "MEMORY CARD", an "RF CARD", and a third board whose label is cut off in the scan ("C…"). Inter-board connectors are CN102/CN202 and CN103/CN203; their pin tables are on the schematic.

## I/O ports (book fig. 9, pp. 71-72)

| Port | Read | Write |
|---|---|---|
| 80-8F | CG ROM data | CG ROM address |
| 90-97 | D7 FS (low = vertical blank, used for scrolling), D6 INT (cursor-blink interrupt), D5-D4 unused, D3-D0 B4-B1 (unused) | D7 FSR (cursor interrupt enable), D6 A/G, D5 +1K (VRAM page offset), D4-D2 GM2-GM0, D1-D0 CSS1/CSS0 |
| 98-9F | — | Low address for an I/O cartridge in the pass-through slot |

- Port 90-97 write bits D1/D0 are cut off in the scan. The text calls them GSS0/GSS1; presumably they are the 6847's CSS bits.
- The "+1K" bit remaps 1 KB segments of the three 2 KB RAMs, so the CPU sees the same addresses for every page.
- Fig. 10 shows which RAM areas each mode and page uses.

## Memory map (X-07 User's Guide + book)

| Range | Contents |
|---|---|
| 8000-97FF | 6 KB VRAM (IC203/204/205) |
| 9800-9FFF | Extension-pack control ROM area (bank-switched, 4 areas) |
| A000-AFFF | TV BASIC ROM (IC220). Starts with the signature `love`, which the X-07 searches for at power-up. |

ROM entry points:

- **Initialisation:** A0CAh (RST 1).
- **BASIC commands:** SCREEN AAD2h, SCREEN(X,Y) ABBDh, COLOR AA73h, PSET ABFBh, PRESET ABF6h, POINT AC14h, LINE AC86h, PAINT AD9Bh, CIRCLE AED2h.
- **Internal routines:** LSETCUR A015h, INTC A050h, ECRIRAM A07Ch, CONPUT A08Bh, MODECR A2C2h, RETLCD A355h, VIDEO A374h, PUTCAR A4D1h.
- **Per-mode code:** dispatch table at A9FBh-AA73h; mode subroutines at A631h-A9FAh.

System RAM variables:

| Address | Meaning |
|---|---|
| B6h | Active page |
| B7h | Displayed page |
| D1h | Screen mode |
| 15h | LCD-font flag |
| 4D7h | KANA |
| 4E5h / 4E6h | Foreground / background colour |
| 4E7h | Palette |
| 4C6h / 4C8h / 4CAh | LINE / STEP coordinates |

## X-07 40-pin extension connector (rear)

| Pin | Signal | Pin | Signal | Pin | Signal | Pin | Signal |
|---|---|---|---|---|---|---|---|
| 1 | GND | 11 | /WR | 21 | D7 | 31 | A15 |
| 2 | GND | 12 | /RD | 22 | D6 | 32 | A14 |
| 3 | GND | 13 | KACL | 23 | D5 | 33 | A13 |
| 4 | GND | 14 | /MAB | 24 | D4 | 34 | A12 |
| 5 | /RESET | 15 | /INTA | 25 | D3 | 35 | A11 |
| 6 | BUZZ | 16 | SI | 26 | D2 | 36 | A10 |
| 7 | VEX | 17 | SO | 27 | D1 | 37 | A9 |
| 8 | VEX | 18 | INTR | 28 | D0 | 38 | A8 |
| 9 | ALE | 19 | RSTC | 29 | CLK | 39 | VCC2 |
| 10 | /WAIT | 20 | NMI | 30 | IO/M | 40 | VCC2 |

Bus notes:

- There are no A0-A7 pins. The NSC800 multiplexes AD0-AD7 on D0-D7, and the X-720 latches the low address byte in IC213 (74LS373) on ALE.
- The bus is 8085-style (/RD, /WR, IO/M).

On the X-720 side (schematic CN201):

- GND pins go through R351/R352 (1 Ω).
- VCC2 feeds VCC3 through R250 (120 Ω).
- VEX goes to VB.
- KACL connects through a switch and a 4700 pF capacitor to VCC2. This is probably the reset/power control; verify.
- BUZZ is not yet traced on the schematic. It presumably feeds the SCART sound output (the French unit outputs RGB-SYNC-SOUND); verify.

## Open problems / next steps

1. **ROM dumps.** IC220 (TV BASIC, A000-AFFF) and IC104 (CG ROM) are both needed. No public dumps were found, and MAME does not emulate the X-720. Options:
   - Dump the ROMs from a real unit. The CG ROM can be read from an X-07 through I/O 80-8F with the X-720 attached.
   - Rewrite the firmware. The entry points above give the required API.
2. **RGB output stage for the French version.** It is undocumented, so it needs a new design: MC6847 to RGB, e.g. via a CPLD/FPGA or an analogue matrix as used on the SECAM Thomson/Dragon/Alice machines.
3. **Better scan of figure 11.** The current markings are marginal. Ask Silicium, or check the Technical Guide on ledudu.
4. **Character set.** The tables in the X-720 manual (pp. 47-49: standard, hiragana and LCD sets) could rebuild the CG ROM contents if no dump is found.

## Sources

- X-720 manual (FR): [Canon_X-720_FR.pdf](Canon_X-720_FR.pdf), originally from https://oldskool.silicium.org/calc/x07/documents/x-720.pdf
- Book "Applications en Assembleur dans l'univers du Canon X-07" (Tonic, Arevian, Millet, Rousseau; Éditions Neptune), ch. 6: [../Documentation/Canon_X-07_AppsASM_FR.pdf](../Documentation/Canon_X-07_AppsASM_FR.pdf), originally from https://oldskool.silicium.org/calc/x07/documents/x-07_AppsASM.pdf
- Owner's page (RGB unit): http://pocket.free.fr/html/canon/x-720_e.html
- Silicium X-07 docs and pinouts: https://oldskool.silicium.org/calc/x07/docs.htm, https://oldskool.silicium.org/calc/x07/connecteurs.htm
- X-07 User's Guide (EN): [../Documentation/Canon_X-07_Users_Guide_EN.pdf](../Documentation/Canon_X-07_Users_Guide_EN.pdf), also at https://ketturi.kapsi.fi/wordpress/wp-content/uploads/2017/02/Canon-X-07-Users-guide.pdf
- Technical Guide (service manual), reportedly on https://canonx07.ledudu.com/pockets.asp?type=116&lg=eng
