# Schematic/Board Review — `mic` (USB-C Microphone + 3.5mm Headphone Monitor)

Reviewed: `mic.kicad_sch` / `mic.kicad_pcb` (KiCad 10.0.4). Original review 2026-08-09 15:00; **updated 2026-08-09 16:00** after schematic revisions.

Method: extracted the full component list and netlist with `kicad-cli sch export netlist`, ran `kicad-cli sch erc` and `kicad-cli pcb drc`, and cross-checked every IC's pin usage against its real datasheet (TI PCM2900C/PCM2902C SBFS039, TI TS5A3159A SCDS200F, TI INA826 SBOS361, TI TLV700 SBVS067, PUI Audio AOM-5024L-HD-R). Only one board exists in this folder (project `mic`), so there's a single review file.

## Status of previous findings — all 5 fixed

| # | Issue | Status |
|---|---|---|
| 1 | +5V rail disconnected from USB VBUS | **Fixed.** `J2` VBUS pins, `U1.V+`, `U2.+VS`, `U4.IN` and `R5` are now one net (verified in netlist — no more separate `Net-(U3-VBUS)`). |
| 2 | Headphone coupling caps (C11/C12) too small (1µF) | **Fixed** electrically. C11/C12 now **220µF** — gives ≈3.6Hz corner into 32Ω, plenty of headroom. See new footprint note below though. |
| 3 | RV1 gain pot wired as fixed resistor, no value set | **Fixed.** RV1 is now `10k`, wiper (pin 2) shorted to pin 3 (rheostat wiring), in series with a new `R4 = 470Ω` floor resistor into `U2.RG`. Gain now adjusts from ≈5.8x (15dB) to ≈107x (40.6dB) exactly as discussed — good general-purpose range for speech. |
| 4 | Mute LED/switch control network broken | **Fixed.** `D1` cathode now returns to GND (Earth net); `U1.pin6 (IN)` is now driven directly from the SW1/R2/R3 junction. Switch closed → mic routed to preamp *and* LED lit; switch open → muted *and* LED off. Matches the fix discussed. |
| 5 | C4 (VCCCI decoupling) at 10µF violated TI's <2µF limit | **Fixed.** C4 is now `1µF`, matching C5–C8. |

Nice work — every fix matches both the underlying datasheet constraint and the specific values we discussed.

## New items introduced by the fixes (small, easy to close out)

### 6. R4 has no footprint assigned
The new `R4` (470Ω, the gain-pot floor resistor) has an **empty Footprint field** in the schematic. It won't get a PCB pad/position and will be flagged as missing during BOM/CPL generation. Assign it the same footprint used for your other resistors (`Resistor_THT:R_Axial_DIN0204_L3.6mm_D1.6mm_P2.54mm_Vertical`), or whatever package you're actually placing.

### 7. C11/C12 footprint doesn't match their new 220µF value
Both are still assigned `Capacitor_THT:C_Disc_D3.4mm_W2.1mm_P2.50mm` — a small ceramic/film disc footprint sized for caps up to roughly 100nF–1µF, not 220µF. A real 220µF part will almost certainly be a radial electrolytic (much bigger body, different pad layout) and won't physically match that footprint.

Two things to fix here:
- Swap the footprint to a radial electrolytic can that fits 220µF at your chosen voltage rating (e.g. `Capacitor_THT:CP_Radial_D6.3mm_P2.50mm` or similar — check the actual part's datasheet for exact diameter/pitch).
- If you go electrolytic, consider switching the symbol from `Device:C` (non-polarized) to `Device:CP` (polarized) so the +/- is explicit on the schematic, and orient the **+ terminal toward the codec's VOUTL/VOUTR pin** (C12 pin1 / C11 pin2) — that side sits at ~1.65V DC bias, the jack side sits at ~0V through the headphone, so that's the correct polarity.

## Everything else from the original review still applies

- **PCB layout**: still 64 DRC violations (21 solder-mask bridges, 21 clearance, 10 silk overlap, 8 silk-over-copper, 3 footprint-library, 1 invalid outline) and 66 unrouted nets — `mic.kicad_pcb` hasn't been touched since the schematic edits, so finish routing and re-run DRC once the two footprint items above are sorted (new/changed footprints will change routing needs, especially the bigger C11/C12 cans).
- **Housekeeping**: `AOM-5024L-HD-F-R`, `INA826AID`, and `PCM2902C` still aren't registered in this machine's library table (cosmetic ERC warnings only — symbols are embedded in the .kicad_sch, so nothing is broken, just add them to your lib table so footprint-linking works cleanly elsewhere).
- **What's correctly designed** (unchanged, still holds): USB-C CC pulldowns (5.1k independent per line), D+/D- 22Ω series resistors, crystal network (12MHz, 22pF, 1MΩ feedback), HID pins grounded / SEL0-SEL1 tied high, mic bias resistor (2.2kΩ, exact match to the AOM-5024L datasheet), INA826 pin usage and 2.5V REF bias divider, TLV70033 wiring and cap values, correct L/R-to-Tip/Ring mapping, single common ground net.

## Priority order to fix

1. Assign a footprint to R4 (#6).
2. Pick a real 220µF part and match its footprint on C11/C12, decide polarized vs not (#7).
3. Finish PCB routing and re-run DRC.
