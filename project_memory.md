# MacroSEL Protection System — Project Memory

_Last updated: March 2026. Merges online Claude web memory with local design documents._
_Authoritative local source: `AICRAFT_latchup_detect_recover.pdf` (formal doc v0.1) — supersedes all `.txt` design files._

---

## Purpose & Context

Bach is designing a radiation-hardened macro single-event latchup (macro-SEL) protection system for a space-qualified edge computer targeting Low Earth Orbit missions with a minimum 3-year operational requirement. The system protects COTS components (ARM Cortex-A72 CPUs, AI cores, DDR4 RAM, eMMC storage) built on advanced process nodes across six power rails.

**Core design philosophy:** Hardware-only, autonomous protection using discrete rad-tolerant components. Commercial eFuse ICs were evaluated and found inadequate for the current and voltage requirements — no single commercial IC meets all requirements (RHRPMICL1A, 3DPM0168 fail on voltage range, current rating, or response time), validating the discrete custom approach.

Bach collaborates with one PCB engineer. The project has a client-facing component with cost and timeline considerations.

---

## Design Evolution

| Version | Key Change |
|---------|-----------|
| v1.0 | Initial: per-rail P-FET switches, 5 retries × 100ms, 6-input OR gate |
| v2.1 | RC snubber on gate driver, single master switch |
| v3.1 | IRHLUC7970Z4 FET (later found undersized), OP07 op-amps |
| v3.2 | BUP06CP038F-01 FET, LMP7704-SP quad op-amps, SN54SC logic |
| v3.3 | **Critical fix:** SR latch for fault memory; split 3.3V/5V power domains; fixed retry-reset flaw where timer reset when Buck #1 collapsed |
| v4.0 (txt) | OPA4H838-SEP; single 5V domain; wired-AND bus; 10ms blanking; 9 components |
| **v0.1 (PDF)** | **Formal design document — current authoritative source.** Added POR RC circuit, Q_dis analysis, per-rail gain/BW table, full response time breakdown. Response time: 3.64 µs. |

v3.x used per-rail P-FET switches and LMP7704-SP op-amps on a 3.3V supply. v4.0 is a fundamental redesign: single master P-channel MOSFET on the 5V upstream rail, wired-AND comparator bus, single 5V power domain.

---

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

Rail 4 uses 50 mΩ (changed from 100 mΩ in v3.x, then from the initially proposed 10 mΩ in v4.0): a 10 mΩ shunt at 0.6A trip needs gain 167 → BW 59.9 kHz, rise time 5.84 µs. The 50 mΩ shunt drops gain to 33.3 → BW 300 kHz, rise time 1.16 µs (5× faster), with only 20 mV drop consuming 30% of the ±67.5 mV CPU rail tolerance budget.

### Signal Flow

**1. Shunt Resistor**
- Part: WSL2512R0100FEA, 10 mΩ ±1%, 2W, 2512 SMD, 4-wire Kelvin connection
- Note: consider ±0.1% tolerance for tighter threshold accuracy

**2. Difference Amplifier — Stage 1** (OPA4H838-SEP, 2× quad, TSSOP-14, powered from 5V_IN)
- Topology: 4-resistor difference amplifier. R1 = R3 = 10 kΩ (E96); R2 varies per rail
- Rp protection resistors removed in final design
- All rails target ~988 mV at trip (uniform V_ref simplifies comparator stage)
- OPA4H838-SEP differential input limit ±0.2V — protective diodes may conduct on transients. Mitigated by R1 = 10 kΩ limiting input current to 330 µA max (well within safe continuous rating)

| Rail | R2 | Gain | BW | Rise time (t_rise = 0.35/BW) |
|------|----|------|----|------|
| 1 | 133 kΩ | 13.18 | 759 kHz | 0.46 µs |
| 2 | 133 kΩ | 13.18 | 759 kHz | 0.46 µs |
| 3 | 221 kΩ | 21.90 | 457 kHz | 0.77 µs |
| 4 | 332 kΩ | 33.20 | 301 kHz | 1.16 µs |
| 5 | 332 kΩ | 33.20 | 301 kHz | 1.16 µs |
| 6 | 665 kΩ | 65.89 | 152 kHz | 2.30 µs |

**3. Voltage Reference** (V_ref ≈ 0.996 V)
- R_top = 40.2 kΩ, R_bot = 10 kΩ, from 5V_IN
- **Rails 1 & 2 share one divider** → 5 total V_ref dividers (not 6)
- 100 nF X7R 0402 decoupling cap at each V_ref node, within 2 mm of comparator IN+ pin (~40 Hz LPF, rejects supply noise)
- V_ref tracks supply linearly: ±5% supply variation shifts trip threshold ±7.5% of rated current (acceptable given 150% margin)

**4. Comparator Stage — Stage 2** (TLV1704-SEP, 2× quad, TSSOP-14, open-drain output, 5V supply)
- **IN+ = V_ref (non-inverting); IN- = V_AMP_n (inverting)**
- Normal: V_AMP_n < V_ref → output Hi-Z
- Fault: V_AMP_n > V_ref → output drives LOW
- Propagation delay: 560 ns typical
- No internal hysteresis — external hysteresis is an open item (see below)

**5. Wired-AND Bus**
- All 6 comparator open-drain outputs on one wire; single 10 kΩ pull-up to 5V_IN
- All Hi-Z → GLOBAL_FAULT = HIGH; any LOW → GLOBAL_FAULT = LOW
- During shutdown: V_AMP collapses → comparators release → bus returns HIGH (false "no fault") — this is the fundamental reason an SR latch is required rather than a direct comparator-to-gate connection

**6. Set Latch Logic** (SN54SC4T32-SEP OR gate)
- OR(GLOBAL_FAULT_L, BLANKING) → SET_L
- During blanking: BLANKING = HIGH forces SET_L = HIGH (inactive), masking inrush false faults

**7. Blanking Timer** (SN54SC6T17-SEP Schmitt, R_blank = 1 MΩ, C_blank = 10 nF)
- Triggered by AC-coupled rising edge of ENABLE_GLOBAL (at retry time, ~1 s — not at T=0)
- Capacitor acts as AC short at the edge, then discharges through R_blank over RC = 10 ms
- Window: 12.2 ms (min) to 18.0 ms (max), typ 15.3 ms
- V_initial ≈ 3.5 V (AC-coupled spike, not full 5V) — timing formula uses V_initial as starting point, decaying to zero

**8. SR Latch** (SN54SC4T00-SEP, 2× NAND gates, active-LOW SET_L / RESET_L)
- Fault: SET_L LOW → FAULT_LATCHED HIGH → ENABLE_GLOBAL LOW → FET off
- Holds state while 5V_IN present, even after all downstream rails collapse to 0V
- Power-on state is indeterminate → resolved by Power on Reset circuit (see below)

**9. Retry Timer** (SN54SC6T17-SEP Schmitt, R_timer = 1.0 MΩ, C_timer = 2.2 µF, τ = 2.2 s)
- Q_dis (JANSR2N3700UB NPN, R_lim_b = 10 kΩ, R_lim_c = 100 Ω) shorts C_timer during normal op
- Fault: ENABLE_GLOBAL LOW → Q_dis OFF → C_timer charges toward 5V
- RETRY_PULSE fires when C_timer crosses Schmitt V_T+: **0.56 s (min), 0.79 s (typ), 1.21 s (max)**
- Timing formula: t = RC × ln(VCC / (VCC − V_T+)) — capacitor charges *toward* rail (asymptote = VCC), contrast with blanking which decays *from* spike to zero
- RETRY_PULSE → inverter (U_INV) → RESET_L (active-LOW) → resets SR latch → retry

**10. Power on Reset** (C_por = 100 nF, R_por = 100 kΩ, RC = 10 ms)
- Couples 5V_IN rising edge → INIT_LATCH_STATE HIGH pulse → inverter → RESET_L LOW → latch reset to safe state
- Fallback: if supply has soft start (edge too slow to couple sufficient voltage spike), retry timer takes over after ~0.56–1.21 s automatically
- INIT_LATCH_STATE and RETRY_PULSE share OR gate → combined RESET_L via inverter

**11. Reset Logic** (SN54SC4T32-SEP OR gate, same IC as SET_L)
- OR(RETRY_PULSE, INIT_LATCH_STATE) → inverter → RESET_L

**12. Enable Logic**
- Single inverter gate: ENABLE_GLOBAL = NOT(FAULT_LATCHED)

**13. Gate Driver** (SN54SC6T14-SEP hex Schmitt inverter)
- ENABLE_GLOBAL HIGH → GATE_DRIVE LOW → Vgs = −5V → FET ON
- ENABLE_GLOBAL LOW → GATE_DRIVE HIGH → Vgs = 0V → FET OFF
- R1 = 10 kΩ pull-up to 5V (fail-safe: FET OFF if driver loses power)
- R2 = 100 Ω series gate resistor (limits dV/dt)
- C_GS = 2.5 nF → τ = R2 × C_GS = 250 ns
- Turn-on: ~128 ns to threshold; turn-off: ~229 ns to threshold; worst case (3τ): 750 ns
- R1 and C2 (AC snubber) are electrically in parallel at the gate node; C2 must be physically routed close to FET body, not back to power input — long traces add parasitic inductance that defeats snubber function

**14. Master FET** (BUP06CP038F-01, P-channel, DPAK/TO-252AA)
- V_DSS: −55V; I_D: −35A continuous, −140A pulsed; R_DS(on): 38 mΩ @ Vgs = −4.5V

### Total Fault Response Time: **3.64 µs**

| Stage | Delay |
|-------|-------|
| Amplifier rise (worst case, Rail 6) | 2.30 µs |
| Comparator propagation | 0.56 µs |
| Logic gates (OR + SR latch + inverter) | ~0.03 µs |
| P-FET turn-off (worst case 3τ) | 0.75 µs |

Dominant contributors: amplifier (Rail 6 has highest gain → lowest BW → slowest rise) and FET switching (~84% combined). Logic gates are negligible.

Note: the 10 ms rail collapse time (bulk capacitor discharge) is irrelevant to protection — the circuit has already acted within 3.64 µs.

### Power Domain

Single **5V_IN** domain — all protection circuits always powered from spacecraft bus regardless of downstream rail state. Total: ~6.5 mA @ 5V = **~33 mW** (0.13% of 25W system budget).

| Block | Current |
|-------|---------|
| 2× OPA4H838-SEP | 5.2 mA |
| 2× TLV1704-SEP | 0.3 mA |
| 4× SN54SC logic ICs | 0.1 mA |
| V_ref dividers (5×) | 0.5 mA |
| Wired-AND pull-up | 0.5 mA |

**Decoupling rule:** 100 nF X7R 0402 ceramic cap on every IC power pin, placed as close to package as possible.

OPA4H838-SEP at 5V_IN provides >1.7 V common-mode headroom above the 3.3V rails — eliminates the common-mode constraint that existed in v3.x (LMP7704-SP at 3.3V).

---

## Bill of Materials

| Ref | Part | Function | Qty |
|-----|------|----------|-----|
| Q1 | BUP06CP038F-01 | Master P-FET | 1 |
| U_AMP_A/B | OPA4H838-SEP | Quad op-amp, current sense, TSSOP-14 | 2 |
| U_CMP_A/B | TLV1704-SEP | Quad comparator, open-drain, TSSOP-14 | 2 |
| U_LATCH | SN54SC4T00-SEP | Quad NAND (SR latch) | 1 |
| U_OR | SN54SC4T32-SEP | Quad OR (SET_L + RESET_L logic) | 1 |
| U_SCH | SN54SC6T17-SEP | Hex Schmitt trigger (blanking + retry timer) | 1 |
| U_INV | SN54SC6T14-SEP | Hex Schmitt inverter (gate driver + reset inversion) | 1 |
| Q_dis | JANSR2N3700UB | NPN transistor (retry timer capacitor discharge) | 1 |

U_INV gate usage: gate driver (1), RESET_L inversion of combined OR output (1) = **2 of 6 gates used — 4 gates spare.**
Note: RETRY_PULSE and INIT_LATCH_STATE are first combined by the OR gate (U_OR), then a single inverter drives RESET_L — not two separate inverters.

---

## Radiation Requirements

- Mission: LEO ~400 km, 3-year minimum, ~3–15 krad/year depending on shielding
- SN54SC series logic, TLV1704-SEP comparators: 30 krad TID RLAT, LET 43 MeV·cm²/mg
- **OPA4H838-SEP: 30 krad(Si) RLAT, LET 43 MeV·cm²/mg** _(not 100 krad — that figure was incorrect in earlier txt versions)_
- BUP06CP038F-01 master FET: 30–50 krad TID, LET 46 MeV·cm²/mg
- 30 krad provides comfortable margin for 3-year LEO mission

---

## Open Items

| Item | Status | Notes |
|------|--------|-------|
| Comparator hysteresis | **Open** | TLV1704-SEP has no internal hysteresis; external hysteresis needed for false-trip immunity on wired-AND bus |
| Q_dis qualification level | **Open** | JANSR2N3700UB (JANS-level) evaluated as drop-in for JANTXV2N2222A (JANTXV-level) — electrically equivalent; verify program quality plan requirements |
| OPA4H838-SEP package in docs | **Resolved** | TSSOP-14 confirmed (not CFP-14 as stated in v4.0 open items). Assembly notes, footprint, land pattern corrected |
| Retry pulse simplification | **Resolved in PDF** | RETRY_PULSE → inverter → RESET_L directly; RC differentiator (C_pulse + R_pull) removed as unnecessary |
| U_INV gate usage | **Resolved** | 2 of 6 gates used (gate driver + RESET_L); 4 spare |
| Power-on reset | **Resolved in PDF** | C_por = 100 nF + R_por = 100 kΩ RC approach implemented (replaces earlier pullup-on-Q proposal) |
| LTspice simulation | **Planned** | With component model substitutions for space-grade parts |
| Prototype functional testing | **Planned** | Consumer-grade equivalent components for logic/function verification across all 6 rails |

---

## Prototype Parts Assessment

For bench testing circuit logic and functionality. All 8 parts are usable.

| Prototype Part | Replaces (Space-Grade) | Verdict | Key Notes |
|---|---|---|---|
| IPD380P06NM | BUP06CP038F-01 | ✅ Use | Near-identical: VDS −60V, ID −35A, RDS(on) 38 mΩ, same D-PAK/TO-252 package, Ciss = 2500 pF matches design value |
| OPA4325 (OPA4325) | OPA4H838-SEP | ✅ Use | Same GBW (10 MHz), slew rate (5 V/µs), supply range, TSSOP-14. Higher offset (150 µV vs ±2.25 µV) acceptable for prototype |
| TLV1704 | TLV1704-SEP | ✅ Use | Same part family (commercial vs space-grade). Identical 560 ns propagation delay, open-drain output, TSSOP-14 |
| SN74HC00 (TSSOP-14) | SN54SC4T00-SEP | ✅ Use | Same quad 2-input NAND function, 2–6V supply |
| SN54HC32 (TSSOP-14) | SN54SC4T32-SEP | ✅ Use | Same quad 2-input OR function, 2–6V supply |
| SN74LV6T17 (TSSOP-14) | SN54SC6T17-SEP | ✅ Use | Schmitt thresholds at 5V nearly identical — computed retry delay 0.54–1.18 s (design: 0.56–1.21 s), blanking 11.8–18.8 ms (design: 12.2–18.0 ms). Within ~5% |
| SN74HC14 (TSSOP-14) | SN54SC6T14-SEP | ✅ Use | Only used for logic inversion (gate driver + RESET_L), not RC timing — threshold differences irrelevant |
| BCP56T1 (SOT-223) | JANSR2N3700UB (ceramic UB SMD) | ✅ Use | Both are SMD, **different footprints**: JANSR2N3700UB uses ceramic UB (~3×2.7 mm); BCP56T1 uses SOT-223 (~6.5×7 mm). Prototype PCB must use SOT-223 pads. hFE min 40 at 150 mA vs 100 for space-grade part — marginal at the ~50 mA discharge peak but acceptable for functional verification |

**U_INV (SN54SC6T14-SEP / prototype SN74HC14) gate usage:** 2 of 6 gates used — 4 spare.
- Gate 1: INVERT(ENABLE_GLOBAL) → GATE_DRIVE (P-FET gate driver)
- Gate 2: INVERT(OR output of RETRY_PULSE + INIT_LATCH_STATE) → RESET_L

RETRY_PULSE and INIT_LATCH_STATE are OR-combined first (U_OR), then a single inverter drives RESET_L — confirmed by functional diagram.

**JANSR2N3700UB package note:** The "UB" suffix designates a ceramic surface-mount package (Microchip/Microsemi), not through-hole. It is a surface-mount equivalent to the JEDEC 2N3700. The BCP56T1 (SOT-223) is also SMD but uses a different, larger footprint.

---

## Key Learnings & Principles

**Circuit topology drives math structure.** Timing formulas must reflect actual circuit topology — blanking uses V_initial (AC-coupled spike, ~3.5V) decaying to zero; retry uses Vcc as asymptote charging from 0V. Using the wrong reference voltage produces wrong results even if the formula looks similar.

**Passive RC networks cannot override actively driven nodes.** Power-on reset must act on a node the passive network can actually influence. The RC differentiator couples into RESET_L via an inverter — it does not try to pull an actively driven NAND gate output.

**Wired-AND bus requires SR latch.** When fault occurs and rails collapse, V_AMP drops to 0V, comparators release the bus, GLOBAL_FAULT returns HIGH — this false "no fault" condition would immediately re-enable the FET. The SR latch decouples fault memory from the sensing signal, holding the system off for the full ~1s retry period.

**Blanking fires at retry (~1s), not at power-on.** The AC-coupled edge-differentiator only fires on ENABLE_GLOBAL LOW→HIGH transitions, which occurs at retry time. Power-on sequencing relies on the POR circuit (or fallback retry delay), not the blanking timer.

**Physical routing matters for high-frequency components.** C2 snubber effectiveness depends on proximity to FET gate/source pads. Long PCB traces add parasitic inductance that defeats the snubber function regardless of component value.

**V_ref supply dependence is a known and accepted trade-off.** V_ref tracks 5V_IN linearly while V_AMP is supply-independent, so supply variation shifts trip threshold: ±5% supply → ±7.5% trip current variation. Acceptable given 150% nominal margin.

**Rail 6 dominates response time.** Highest gain (65.89×) → lowest BW (152 kHz) → slowest amplifier rise time (2.30 µs). This is the worst-case figure used in total system response time.

**Cross-checking against source documents is essential.** Multiple documented instances where Claude responses contained errors (pin designations, component ratings, version-specific data). Bach catches these by referencing the actual design document and expects explicit acknowledgment and correction, not hedged language. Errors to watch for: mixing v3.x and v4.0 specs, incorrect OPA4H838-SEP radiation figures (30 krad, not 100 krad).

---

## Approach & Patterns

- Bach actively cross-references Claude's technical claims against source documents and directly challenges errors, expecting explicit correction rather than hedged language.
- Design decisions are documented with written technical justifications that Bach drafts and submits for review — Claude's role includes identifying factual errors, logic errors, and imprecise language in those drafts.
- Analysis proceeds component by component and signal-path by signal-path, with careful attention to which design version is being discussed — v3.x and v4.0 data must not be mixed.
- Bach demonstrates strong circuit intuition and frequently arrives at correct conclusions independently, using Claude to verify reasoning and fill in derivations.

---

## Tools & Resources

- **Authoritative design doc:** `AICRAFT_latchup_detect_recover.pdf` (formal v0.1) — supersedes `MacroSEL_Protection_v4_0_FINAL.txt`
- **Diagrams:** `latchup_detect_recover.drawio` (source) + `latchup_detect_recover-*.drawio.png` (exports per sub-block)
- **Datasheets:** `./latchup_v4.0/` (space-grade parts); `./latchup_v4.0/prototype_commercial_parts/` (commercial equivalents for prototyping)
- **Application notes:** `snoaa71.pdf` (space-grade 3-op-amp instrumentation amplifier circuit), `01332b.pdf` (current sensing fundamentals)
- **Simulation:** LTspice (planned, with component model substitutions)
- **Key components:** OPA4H838-SEP, TLV1704-SEP, SN54SC4T00-SEP, SN54SC6T17-SEP, SN54SC6T14-SEP, BUP06CP038F-01, JANSR2N3700UB, WSL2512R0100FEA shunt resistors
