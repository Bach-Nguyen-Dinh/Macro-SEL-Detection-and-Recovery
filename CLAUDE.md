# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **Primary memory file: `project_memory.md`** — read this first for full project context, design decisions, open items, key learnings, and version history. This CLAUDE.md summarises the stable architecture for quick reference only.

at directory ./latchup_v4.0 are part used for the design. inside the ./latchup_v4.0 directory, there is another directory ./latchup_v4.0/prototype_commercial_parts contains parts to build a prototype for testing the logic and functionality of the design.

## Project Overview

This is a **hardware design** repository (not a software project) for a radiation-tolerant Macro Single Event Latchup (MacroSEL) protection system. The system protects an ARM Cortex-A72 Edge Computer in LEO orbit (3-year minimum mission) against single-event latchup events on its power rails.

Design philosophy: **hardware-only autonomous protection** — no firmware or supervisory controller. Every component must justify its existence.

## Document Hierarchy (newest → oldest)

| File | Version | Status |
|------|---------|--------|
| `AICRAFT_latchup_detect_recover.pdf` | v0.1 (formal doc) | **Authoritative — supersedes all txt files** |
| `MacroSEL_v4_Complete.docx` | v4.0 | Supporting reference |
| `MacroSEL_Protection_v4_0_FINAL.txt` | v4.0 | Superseded by PDF |
| `MacroSEL_Protection_v3_3_CORRECTED.txt` | v3.3 | History only |
| `MacroSEL_Protection_v3_2_Complete.txt` | v3.2 | History only |
| `MacroSEL_Protection_FINAL_v2.1_RC_Snubber.txt` | v2.1 | History only |
| `MacroSEL_Protection_Complete_Design.txt` | v1.0 | History only |

The `AICRAFT_latchup_detect_recover*.docx` files are working copies of the same formal document.

## Repository Structure

- `latchup_detect_recover-*.drawio.png` — Circuit block diagrams exported from draw.io
- `latchup_detect_recover.drawio` — draw.io source file
- `latchup_v4.0/` — Component datasheets for current design (authoritative)
- `latchup_v4.0/application_notes/` — Referenced application notes (snoaa71.pdf, 01332b.pdf)
- `latchup_v4.0/prototype_commercial_parts/` — Commercial equivalents for lab prototyping
- `latchup_v3.1/` — Datasheets for older design revision (historical)

## System Architecture

### Protected Power Rails
Trip threshold: **150% of rated current** on each rail.

| Rail | Voltage | Rated Current | Load | Shunt | Trip Current |
|------|---------|---------------|------|-------|-------------|
| 1 | 3.3V | 5A | AI Core #1 | 10 mΩ | 7.5A |
| 2 | 3.3V | 5A | AI Core #2 | 10 mΩ | 7.5A |
| 3 | 1.2V | 3A | CPU + DDR4 | 10 mΩ | 4.5A |
| 4 | 1.35V | 400mA | CPU | 50 mΩ | 0.6A |
| 5 | 1.8V | 2A | CPU + eMMC | 10 mΩ | 3.0A |
| 6 | 2.5V | 1A | CPU + DDR4 | 10 mΩ | 1.5A |

Rail 4 uses 50 mΩ (not 10 mΩ) because: a 10 mΩ shunt would need gain 167 → only 59.9 kHz BW; 50 mΩ at gain 33.3 gives 300 kHz BW and 1.16 µs rise time (5× faster detection), with only 20 mV voltage drop consuming 30% of the ±67.5 mV rail tolerance budget.

### Signal Flow

1. **Shunt resistor** (WSL2512R0100FEA, 10 mΩ ±1%, 2W, 2512 SMD, 4-wire Kelvin) converts rail current to voltage.

2. **Stage 1 — Difference Amplifiers** (OPA4H838-SEP, 2× quad, TSSOP-14, 5V supply from 5V_IN):
   - Fixed R1 = 10 kΩ; R2 varies per rail to target ~988 mV at trip
   - IN+/IN- differential input limit ±0.2V — handled safely by 10 kΩ R1 limiting input current to 330 µA max
   - Differential input limit ±0.2V is a known concern during transients; current-limiting via R1 (10 kΩ) is the mitigation

   | Rail | R2 | Gain | BW | Rise time |
   |------|----|------|----|-----------|
   | 1 | 133 kΩ | 13.18 | 759 kHz | 0.46 µs |
   | 2 | 133 kΩ | 13.18 | 759 kHz | 0.46 µs |
   | 3 | 221 kΩ | 21.90 | 457 kHz | 0.77 µs |
   | 4 | 332 kΩ | 33.20 | 301 kHz | 1.16 µs |
   | 5 | 332 kΩ | 33.20 | 301 kHz | 1.16 µs |
   | 6 | 665 kΩ | 65.89 | 152 kHz | 2.30 µs |

3. **Voltage Reference** (V_ref ≈ 0.996 V): R_top = 40.2 kΩ, R_bot = 10 kΩ from 5V_IN. **Rails 1 & 2 share one divider** → 5 total dividers. Each V_ref node has a 100 nF X7R 0402 cap to GND within 2 mm of comparator pin (~40 Hz LPF to reject supply noise).
   - V_ref tracks supply linearly: ±5% supply variation shifts trip threshold by ±7.5% of rated current.

4. **Stage 2 — Comparators** (TLV1704-SEP, 2× quad, TSSOP-14, open-drain output, 5V supply):
   - IN+ = V_ref; IN- = V_AMP_n
   - Normal: V_AMP_n < V_ref → output Hi-Z
   - Fault: V_AMP_n > V_ref → output drives LOW
   - Propagation delay: 560 ns typical

5. **Wired-AND bus**: All 6 comparator open-drain outputs tied to one wire with single 10 kΩ pull-up to 5V_IN.
   - All Hi-Z → GLOBAL_FAULT = HIGH (no fault)
   - Any LOW → GLOBAL_FAULT = LOW (fault)
   - During shutdown: V_AMP collapses → comparators release → bus returns HIGH (false "no fault") — this is why an SR latch is required, not direct connection.

6. **Set Latch Logic** (SN54SC4T32-SEP OR gate): OR(GLOBAL_FAULT_L, BLANKING) → SET_L. During blanking window, BLANKING = HIGH forces SET_L = HIGH (inactive), masking inrush false faults.

7. **Blanking Timer** (SN54SC6T17-SEP Schmitt, R_blank = 1 MΩ, C_blank = 10 nF):
   - Triggered by rising edge of ENABLE_GLOBAL coupling through C_blank
   - Window: 12.2 ms (min) to 18.0 ms (max), typ 15.3 ms

8. **SR Latch** (SN54SC4T00-SEP, 2× NAND gates, active-LOW inputs SET_L / RESET_L):
   - Fault detected → SET_L goes LOW → FAULT_LATCHED = HIGH → ENABLE_GLOBAL = LOW → FET off
   - Holds state while 5V_IN is present, even after all downstream rails collapse to 0V

9. **Retry Timer** (SN54SC6T17-SEP Schmitt, R_timer = 1.0 MΩ, C_timer = 2.2 µF, τ = 2.2 s):
   - Q_dis (JANSR2N3700UB NPN, R_lim_b = 10 kΩ, R_lim_c = 100 Ω) shorts capacitor during normal operation
   - On fault: ENABLE_GLOBAL = LOW → Q_dis OFF → C_timer charges toward 5V
   - RETRY_PULSE fires when C_timer crosses Schmitt V_T+: 0.56 s (min) to 1.21 s (max), typ 0.79 s
   - RETRY_PULSE → inverter → RESET_L (active-LOW) → resets SR latch → system retries

10. **Power on Reset** (C_por = 100 nF, R_por = 100 kΩ, RC = 10 ms):
    - Couples 5V_IN rising edge → INIT_LATCH_STATE HIGH pulse → inverter → RESET_L LOW → latch reset
    - Fallback: if power rise is soft, retry timer takes over after ~1 s

11. **Reset Logic** (SN54SC4T32-SEP OR gate, shared IC with SET_L): OR(RETRY_PULSE, INIT_LATCH_STATE) → feeds inverter → RESET_L

12. **Enable Logic**: Single inverter — ENABLE_GLOBAL = NOT(FAULT_LATCHED)

13. **Gate Driver** (SN54SC6T14-SEP inverter):
    - ENABLE_GLOBAL HIGH → GATE_DRIVE LOW → Vgs = −5V → FET ON
    - ENABLE_GLOBAL LOW → GATE_DRIVE HIGH → Vgs = 0V → FET OFF
    - R1 = 10 kΩ pull-up to 5V (fail-safe OFF if driver loses power)
    - R2 = 100 Ω series, C_GS = 2.5 nF → τ = 250 ns; turn-on ~128 ns, turn-off ~229 ns, worst-case 750 ns

14. **Master FET** (BUP06CP038F-01, P-channel, DPAK/TO-252AA, −35A continuous, −140A pulsed, R_DS(on) 38 mΩ @ Vgs = −4.5V)

### Total Fault Response Time: 3.64 µs

| Stage | Delay |
|-------|-------|
| Amplifier rise (worst case, Rail 6) | 2.30 µs |
| Comparator propagation | 0.56 µs |
| Logic gates (OR + SR latch + inverter) | ~0.03 µs |
| P-FET turn-off (worst case 3τ) | 0.75 µs |

### Power Domain

Single **5V_IN** domain — all protection circuits always powered from the spacecraft bus. Total consumption: ~6.5 mA @ 5V = ~33 mW (0.13% of 25W system budget).

| Block | Current |
|-------|---------|
| 2× OPA4H838-SEP | 5.2 mA |
| 2× TLV1704-SEP | 0.3 mA |
| 4× SN54SC logic ICs | 0.1 mA |
| V_ref dividers (5×) | 0.5 mA |
| Wired-AND pull-up | 0.5 mA |

**Decoupling**: 100 nF X7R 0402 ceramic cap on each IC power pin, placed as close to package as possible.

## Bill of Materials (Current Design)

| Ref | Part | Function | Qty |
|-----|------|----------|-----|
| Q1 | BUP06CP038F-01 | Master P-FET | 1 |
| U_AMP_A/B | OPA4H838-SEP | Quad op-amp (current sense) | 2 |
| U_CMP_A/B | TLV1704-SEP | Quad comparator | 2 |
| U_LATCH | SN54SC4T00-SEP | Quad NAND (SR latch) | 1 |
| U_OR | SN54SC4T32-SEP | Quad OR (SET_L + RESET_L logic) | 1 |
| U_SCH | SN54SC6T17-SEP | Hex Schmitt (blanking + retry timer) | 1 |
| U_INV | SN54SC6T14-SEP | Hex Schmitt inverter (gate driver + reset logic) | 1 |
| Q_dis | JANSR2N3700UB | NPN transistor (retry timer discharge) | 1 |

Shunt resistors: WSL2512R0100FEA (10 mΩ ±1%, 2W) — consider ±0.1% tolerance.

## Radiation Requirements

- Mission: LEO ~400 km, 3-year minimum, ~3–15 krad/year depending on shielding
- All ICs: ≥30 krad TID (SN54SC series, TLV1704-SEP: 30 krad RLAT, LET 43 MeV·cm²/mg)
- Op-amps (OPA4H838-SEP): **30 krad(Si) RLAT**, LET 43 MeV·cm²/mg
- Master FET (BUP06CP038F-01): 30–50 krad TID, LET 46 MeV·cm²/mg

## draw.io Diagrams

The `latchup_detect_recover-*.drawio.png` files correspond to sub-blocks:
- `FUNCTIONAL_DIAGRAM` — top-level system overview
- `current_sense_amplifier` — Stage 1 difference amplifier
- `comparator` — Stage 2 comparator with open-drain output
- `Wired_AND_bus` — how comparator open-drain outputs combine
- `NAND_SR_latch` — fault state memory
- `power_on_reset` / `delay_startup` — startup blanking and POR
- `delay_retry` / `retry_pulse_gen` — ~1s retry timer
- `set_fault_latched` — latch set/reset OR logic
- `gate_driver` / `gate_driver_direct_latch_out` — P-FET gate control
- `reference_voltage` — V_ref resistor divider from 5V_IN

## Version History Summary

| Version | Key Change |
|---------|-----------|
| v1.0 | Initial: 5 retries × 100ms, 6-input OR gate aggregation |
| v2.1 | RC snubber on gate driver, single master switch |
| v3.1 | Replaced IRHLUC7970Z4 FET (undersized), used OP07 op-amps |
| v3.2 | BUP06CP038F-01 FET, LMP7704-SP quads, SN54SC logic series |
| v3.3 | **Critical fix**: SR latch added for fault memory; split 3.3V/5V power domains; fixed retry-reset flaw |
| v4.0 (txt) | OPA4H838-SEP; single 5V domain; wired-AND bus; 10ms blanking; 9 ICs |
| v0.1 (PDF) | Formal design document; added POR circuit, Q_dis details, response time analysis, per-rail gain/BW table |
