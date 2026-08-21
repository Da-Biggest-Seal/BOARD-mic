# Schematic/Board Review — `mic` (USB-C Microphone + 3.5mm Headphone Monitor)

Reviewed: `mic.kicad_sch` / `mic.kicad_pcb` (KiCad 10.0.4). Original review 2026-08-09; updated through 2026-08-21 across several passes (datasheet re-verification, a C9/VCOM routing episode that resolved cleanly); **updated again 2026-08-21 (18:40)** after you added a REF bypass cap and asked for a value check on C11/C12. Only one board exists in this folder (project `mic`), so there's a single review file.

Method: exported the netlist (`kicad-cli sch export netlist`), ran `kicad-cli sch erc` / `kicad-cli pcb drc`, and verified IC pin usage, gain/decoupling math, and reference circuits against the actual datasheet PDFs: TI PCM2900C/PCM2902C SBFS039, TI TS5A3159A SCDS200F, TI INA826 SBOS562G, TI TLV700 SLVSA00E, and PUI Audio AOM-5024L-HD-R.

## Current state: clean

PCB routing (including the earlier C9/VCOM episode) is fully resolved — 0 `clearance`/`hole_clearance`/unconnected issues. With all 12 normally-ignored DRC categories restored to `"warning"` for a full scratch-copy check, the board sits at the same 12 minor cosmetic items as the original clean-routing pass (table below). ERC: 7 cosmetic warnings, 0 errors.

## Status of the previous review's findings

| # | Issue | Status |
|---|---|---|
| 1 | VCCCI decoupling (`C4`) was 1µF, should be 10µF per TI's reference circuit | **Fixed.** `C4` is now 10µF. `C5–C8` correctly stay at 1µF (VCCP1I/VCCP2I/VCCXI/VDDI, TI's <2µF pins). |
| 2 | `R9` (2.2kΩ) sat in series between VCOM and its bypass cap `C9`, which TI's reference circuit doesn't do | **Fixed**, schematic and PCB now agree, and the PCB clearance issues that appeared partway through fixing this are resolved as of the latest edit (see above). |
| 3 | INA826 `REF` pin fed by an unbuffered 10k/10k divider (R5/R6), no bypass cap — TI requires REF to be driven by low impedance to preserve CMRR | **Fixed.** New `C15 = 4.7µF` added from REF to GND. Within the recommended 1–10µF bypass range — good choice. |
| 4 | C11/C12 (220µF headphone coupling caps) still using a small-disc footprint and non-polarized `Device:C` symbol sized for ≤1µF, not 220µF | **Value confirmed OK, footprint/symbol still to update** — see part B below for the capacitance analysis you asked for. |
| 5 | 12 DRC rule categories set to `"ignore"` in `mic.kicad_pro` | **Acknowledged as intentional** — you've said you know about this and it's fine, so it's no longer tracked as an open item here. Only flagging again if the DRC picture materially changes. |

## Findings still open — detail

### A. INA826 REF pin — closed
`C15` (4.7µF, REF → GND) resolves this. REF now has a proper AC bypass close to the pin instead of a bare unbuffered divider, which should meaningfully help the preamp's CMRR and noise floor.

### B. C11/C12 — is 220µF the right value? (you asked)
Short answer: **it depends on what headphones actually get plugged in, and 220µF is on the small side for typical consumer headphones/earbuds.** The AC-coupling corner frequency is `f_c = 1/(2πRC)`, where R is the headphone's own driver impedance (there's no headphone amp on this board — C11/C12 couple straight from the codec's VOUTL/VOUTR into the jack, so the headphone itself is the load the cap sees):

| Headphone impedance | 220µF (current) | 330µF | 470µF | 1000µF |
|---|---|---|---|---|
| 16Ω (earbuds, worst case) | 45.2 Hz | 30.1 Hz | 21.2 Hz | 10.0 Hz |
| 32Ω (typical consumer) | 22.6 Hz | 15.1 Hz | 10.6 Hz | 5.0 Hz |
| 150Ω (higher-Z cans) | 4.8 Hz | — | — | — |
| 300Ω (high-Z studio) | 2.4 Hz | — | — | — |

A single-pole RC high-pass is only ~1dB down at 2×f_c, so you want f_c comfortably below 20Hz (the bottom of hearing) for it to be inaudible. At 220µF: fine into 150–300Ω cans (2–5Hz), but into a typical 32Ω headphone the corner sits at 22.6Hz — *above* 20Hz, meaning you'd roll off some of the lowest audible bass; into 16Ω earbuds (45Hz) it's a clearly audible bass cut.

Since this is a general-purpose monitoring jack where you don't control what someone plugs in, and most consumer headphones/earbuds sit in the 16–32Ω range: **I'd go up to at least 470µF, ideally 1000µF**, rather than keep 220µF. Both are common, cheap radial-electrolytic values, so it doesn't cost you anything to size up before you order parts.

One related note while you're at it, not something to fix, just context for expectations: the PCM2902C's own DAC spec (0.005% THD, 96dB SNR) is characterized into a 10kΩ AC-coupled load, not a 16–32Ω headphone — TI's own reference circuits show it feeding a separate headphone-amp block (`LPF, Amp` in Figures 36–39), not driving the jack directly. Your design skips that amp and drives the jack straight from VOUTL/VOUTR, same as a lot of cheap USB "sound card" dongles do. It'll work, just expect lower max volume and somewhat more distortion into low-impedance headphones than the datasheet's headline numbers — not a defect, just worth knowing going in.

**Fix:** pick your real part (470µF or 1000µF, radial electrolytic, rated ≥6.3V is plenty since the bias is ~1.65V), then switch the symbol to `Device:CP` (polarized) and update the footprint to match its actual body size (e.g. `Capacitor_THT:CP_Radial_D6.3mm_P2.50mm` or larger — check the real part's datasheet). Orient + toward the codec side (C12 pin1 / C11 pin2, ~1.65V DC bias) since the jack side sits near 0V through the headphone.

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

1. Decide on C11/C12's real value (470µF or 1000µF recommended over 220µF — see part B above) and order polarized electrolytics sized accordingly.
2. Update C11/C12's symbol (`Device:CP`) and footprint once the real part is picked.
3. Clean up the 12 minor silkscreen/footprint/edge-clearance items (table above) before fab, whenever convenient — none are urgent.
