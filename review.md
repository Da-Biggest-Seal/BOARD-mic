# Schematic/Board Review — `mic` (USB-C Microphone + 3.5mm Headphone Monitor)

Reviewed: `mic.kicad_sch` / `mic.kicad_pcb` (KiCad 10.0.4). Original review 2026-08-09; updated through 2026-08-21 across several passes (datasheet re-verification, a C9/VCOM routing episode, a REF-pin fix, a capacitance sanity check); **updated again 2026-08-21 (19:20)** after you sized C11/C12 up to 470µF with a real radial-electrolytic footprint. Only one board exists in this folder (project `mic`), so there's a single review file.

Method: exported the netlist (`kicad-cli sch export netlist`), ran `kicad-cli sch erc` / `kicad-cli pcb drc`, and verified IC pin usage, gain/decoupling math, and reference circuits against the actual datasheet PDFs: TI PCM2900C/PCM2902C SBFS039, TI TS5A3159A SCDS200F, TI INA826 SBOS562G, TI TLV700 SLVSA00E, and PUI Audio AOM-5024L-HD-R.

## Current state: all real electrical/value findings closed, one cosmetic side-effect to clean up

PCB routing is fully resolved — 0 `clearance`/`hole_clearance`/unconnected issues, ERC 7 cosmetic warnings / 0 errors. `C11`/`C12` are now `470µF` on `Capacitor_THT:CP_Radial_D10.0mm_P5.00mm` (polarized, correctly sized) — exactly the fix from the capacitance check. That's every substantive finding from this review now closed.

The one side effect: swapping in the physically much bigger 470µF cans pushed silkscreen-overlap violations from 1 up to 33 (full-severity DRC count now 44, up from 12) — almost all of it is `C11`'s own reference-designator text sitting on top of its new, much larger silkscreen outline, plus a few overlaps with neighboring `U1`'s designator. **Not a physical collision** — I checked: `C11` (183.9, 58.75) sits ~8.3mm from `U1` (176.9, 63.3), and there's no `courtyards_overlap` violation, so the parts don't actually touch, it's just crowded silkscreen text. Worth nudging `C11`'s (and `C12`'s) reference designator to a clear spot before ordering silkscreen, but it's cosmetic, not electrical.

## Status of the previous review's findings

| # | Issue | Status |
|---|---|---|
| 1 | VCCCI decoupling (`C4`) was 1µF, should be 10µF per TI's reference circuit | **Fixed.** `C4` is now 10µF. `C5–C8` correctly stay at 1µF (VCCP1I/VCCP2I/VCCXI/VDDI, TI's <2µF pins). |
| 2 | `R9` (2.2kΩ) sat in series between VCOM and its bypass cap `C9`, which TI's reference circuit doesn't do | **Fixed**, schematic and PCB now agree, and the PCB clearance issues that appeared partway through fixing this are resolved as of the latest edit (see above). |
| 3 | INA826 `REF` pin fed by an unbuffered 10k/10k divider (R5/R6), no bypass cap — TI requires REF to be driven by low impedance to preserve CMRR | **Fixed.** New `C15 = 4.7µF` added from REF to GND. Within the recommended 1–10µF bypass range — good choice. |
| 4 | C11/C12 (headphone coupling caps) undersized at 220µF for typical low-impedance headphones, plus wrong footprint/symbol | **Fixed.** Now 470µF, `Device:CP` polarized, `CP_Radial_D10.0mm_P5.00mm` footprint. See part B below for the math this was based on. |
| 5 | 12 DRC rule categories set to `"ignore"` in `mic.kicad_pro` | **Acknowledged as intentional** — you've said you know about this and it's fine, so it's no longer tracked as an open item here. Only flagging again if the DRC picture materially changes. |

## Findings still open — detail

### A. INA826 REF pin — closed
`C15` (4.7µF, REF → GND) resolves this. REF now has a proper AC bypass close to the pin instead of a bare unbuffered divider, which should meaningfully help the preamp's CMRR and noise floor.

### B. C11/C12 capacitance — closed
Original question: is 220µF the right value? Answer was no — it depends on what headphones actually get plugged in, and 220µF was on the small side for typical consumer headphones/earbuds. The AC-coupling corner `f_c = 1/(2πRC)` uses the headphone's own driver impedance as R (no headphone amp on this board — C11/C12 couple straight from VOUTL/VOUTR into the jack):

| Headphone impedance | 220µF (old) | 470µF (now) | 1000µF |
|---|---|---|---|
| 16Ω (earbuds, worst case) | 45.2 Hz | 21.2 Hz | 10.0 Hz |
| 32Ω (typical consumer) | 22.6 Hz | 10.6 Hz | 5.0 Hz |
| 150Ω (higher-Z cans) | 4.8 Hz | 2.3 Hz | — |

At 220µF the corner into a typical 32Ω headphone sat above 20Hz — audible bass rolloff. At the new 470µF it's 10.6Hz into 32Ω / 21.2Hz into 16Ω earbuds — a solid improvement, comfortably inaudible for most headphones, only mildly tight for the worst-case 16Ω earbud scenario. Good choice; 1000µF would be marginally safer still but 470µF is a reasonable, common value to land on.

One related note, not something to fix, just context: the PCM2902C's own DAC spec (0.005% THD, 96dB SNR) is characterized into a 10kΩ AC-coupled load — TI's reference circuits show it feeding a separate headphone-amp block, not driving the jack directly. This design drives the jack straight from VOUTL/VOUTR, same as a lot of cheap USB "sound card" dongles do. It'll work, just expect lower max volume and somewhat more distortion into low-impedance headphones than the datasheet's headline numbers.

## Library housekeeping (cosmetic, nothing electrically broken)
`AOM-5024L-HD-F-R`, `INA826AID`, and `PCM2902C` still aren't registered in this machine's footprint/symbol library tables. Symbols/footprints are embedded in the files so the board itself isn't broken by this, but add them to your library tables for portability.

## What's confirmed correct (re-verified against the real datasheets)

- **PCM2902C**: every pin used (VBUS/D±/XTI/XTO/VINL/VINR/VOUTL/VOUTR/VCOM/SEL0/SEL1/HID0-2/DIN) matches the real SBFS039 pin table. SEL0/SEL1 tied to +3.3V is a valid logic-high. HID0-2 correctly grounded. DIN correctly grounded (S/PDIF unused). Crystal network (Y1=12MHz, C13/C14=22pF, R10=1MΩ) matches the reference circuit. C10=4.7µF VIN coupling matches TI's analog front-end diagram. VOUTL→Tip, VOUTR→Ring is the correct TRS mapping.
- **TS5A3159A (mute switch)**: pin table verified against the actual datasheet image (SOT-23-6: NO=1, GND=2, NC=3, COM=4, V+=5, IN=6). Mute logic traced end-to-end and correct: switch open → muted + LED off; switch closed → mic connected to INA826 +IN + LED on. R11 (10k pulldown) correctly keeps the preamp input from floating while muted.
- **INA826**: pin table confirmed; gain equation G = 1 + 49.4kΩ/RG confirmed. RV1 (10k rheostat) + R4 (470Ω floor) gives RG = 470Ω–10.47kΩ → gain 5.72×–106× (15.1–40.5dB), a good range for speech.
- **TLV70033**: pin table confirmed (IN=1, GND=2, EN=3, NC=4, OUT=5). EN tied to +5V — correct. C2/C3 = 1µF on IN/OUT matches the datasheet.
- **AOM-5024L-HD-R capsule**: R1 = 2.2kΩ matches PUI Audio's spec'd load resistor exactly. Biased from the regulated +3.3V rail rather than raw +5V — keeps USB bus noise out of the most sensitive node, and 3.3V is close to the capsule's 3V rated voltage.
- **USB-C**: CC1/CC2 each get their own independent 5.1kΩ pulldown (R12/R13) — correct per spec. D+/D- 22Ω series resistors match TI's reference circuit.
- **Ground**: single common "Earth" net; all codec ground pins commoned as recommended.

## Priority order to fix

Nothing electrical or value-related is left open. What remains is cosmetic PCB cleanup before you order fab:

1. Nudge `C11`/`C12`'s reference-designator text off their new (much larger) silkscreen outline, and clear of `U1`'s designator nearby — this is the bulk of the 33 `silk_overlap` hits from sizing the caps up to 470µF. No physical part collision (courtyards don't overlap), purely a legend/readability issue.
2. Whenever convenient, sweep the rest: a couple of `silk_over_copper`/`silk_edge_clearance` hits near MK2/SW1, a `lib_footprint_mismatch` on J1 worth a quick diff against the library copy, 3 `copper_edge_clearance` warnings near J2/SW1 worth a fab-tolerance check, and the library-table registration gaps for AOM-5024L-HD-F-R/INA826AID/PCM2902C — all cosmetic, none urgent.
