# 10-BJT-VCCS-Diagnosis

## 🔑 TL;DR

**Precision ±10–12.5A analog current driver for magnetic bias coils in a cold-atom/MOT physics experiment, sub-100µs response.**

- 🔧 **Diagnosed & recovered** a dead legacy MOSFET/IGBT current-source prototype — traced a failed IGBT gate driver, resolved ground-loop and thermal issues
- 🔁 **Redesigned from scratch**: precision op-amp (OPA455) + discrete BJT push-pull output stage, replacing the old MOSFET/IGBT topology
- 🛡️ **Added a 5-diode protection network** to prevent transistor degradation from reverse-bias stress during power cycling
- 🚧 **Status:** board fabricated, currently in bring-up — actively debugging an open 5A idle-current issue (ground loop vs. bias runaway)

---

![https://github.com/DavodChuweiLiao/10-BJT-VCCS-Diagnosis/blob/main/Layout.png?raw=true](https://github.com/DavodChuweiLiao/10-BJT-VCCS-Diagnosis/blob/main/Layout.png?raw=true)
![https://github.com/DavodChuweiLiao/10-BJT-VCCS-Diagnosis/blob/main/Screenshot%202026-09-14%20211057.png?raw=true](https://github.com/DavodChuweiLiao/10-BJT-VCCS-Diagnosis/blob/main/Screenshot%202026-09-14%20211057.png?raw=true)
![https://github.com/DavodChuweiLiao/10-BJT-VCCS-Diagnosis/blob/main/actualboard.jpg?raw=true](https://github.com/DavodChuweiLiao/10-BJT-VCCS-Diagnosis/blob/main/actualboard.jpg?raw=true)

# Precision Bias Coil Current Driver — Project README

**Owner:** Chuwei (David) Liao
**System purpose:** Generate stable, fast-switching, low-ripple current through magnetic bias coils (X, Y, Z, and MOT) for a cold-atom / magneto-optical trap experiment. Command is an analog setpoint voltage; each axis needs bidirectional current up to ~10-12.5A with sub-100µs response.

This document consolidates the project's full history — the original prototype hardware, the recovery/debug log after it was found not working, and the new linear amplifier redesign — into a single reference.

---

## 1. Timeline Summary

| Phase | Date | Description |
|---|---|---|
| Original thesis | — | Mithilesh Kumar Parit's PhD thesis establishes the target performance (500µs response @ 1A/2A steps, 4-axis Helmholtz coil driving) |
| Prototype build | documented July 7, 2026 | 4-unit enclosure (X/Y/Z/MOT), MOSFET+IGBT linear VCCS topology, breadboard construction |
| Recovery / debug pass | 2026 (Evernote log) | Circuit recovered from storage; Y-axis smoke incident, X-axis IGBT driver found dead, response times re-measured, low-side redesign explored |
| New amplifier redesign | current | OPA455-based op-amp + discrete complementary BJT push-pull (EF2) output stage, replacing the MOSFET/IGBT linear-VCCS approach; PCB fabricated and under bring-up |

---

## 2. Phase 1 & 2 — Original Prototype ("Current Stabilization Circuit")

### 2.1 Architecture
High-power linear **Voltage-Controlled Current Source (VCCS)**. A series MOSFET operates in its linear region as a voltage-controlled variable resistor; an external analog PID (SRS LTS-series controller) closes the loop around a current transducer measurement to hold current constant regardless of coil impedance or thermal drift. No PWM switching noise — pure linear control.

**Signal path:** `+30V supply → high-side MOSFET (drain) → series IGBT (static ON/OFF switch) → coil load → LEM current transducer → PGND`. Transducer secondary (1:250 ratio, 4mA per 1A primary) drives a 100Ω burden resistor, producing 0.4V/A feedback to the external PID. PID output drives the MOSFET gate directly (unbuffered).

### 2.2 Hardware Configuration
- 4 physical units in one metal enclosure: **X-Bias, Y-Bias, Z-Bias, MOT**
- Through-hole breadboard construction, hand-wired
- BNC connectors for signals (Feedback, Measure, Setpoint, PWM, DISABLE); banana plugs for power
- 2 cooling fans; IGBT gate drivers mounted on enclosure walls

### 2.3 Power/Logic Inputs (per axis)
- **Main Power (Red/Black):** +5V to +30V drive rail
- **12V Logic:** for internal circuitry
- **5V Logic:** for internal circuitry — ⚠️ swapping 5V/12V or reversing polarity destroys the IGBT driver
- **±15V Bipolar:** powers the analog current transducers (can be synthesized from two isolated single-channel supplies, tied at a common GND point)
- **As-built quirk:** Z-Bias and MOT share a single ±15V line for their transducers — do not wire a second, conflicting ±15V supply to either.

### 2.4 As-Built Grounding Architecture (critical — caused real issues)
- On **X-Bias and Z-Bias only**: Main Power return (Power−) is internally tied to 12V logic ground.
- On **all axes**: the external "Measure" ground is internally tied to the current transducer's internal ground.
- Because of the above, external power supplies **must have floating outputs** — otherwise an earth-ground path plus the internal ties creates a ground loop.

### 2.5 Absolute Maximum Ratings
| Parameter | Limit |
|---|---|
| Max load current | **12.5A** (hard limit of the current transducer) |
| Max main power voltage | **100V** |
| Logic rails | Strictly 5V / 12V / ±15V — over-voltage destroys front-end ICs and IGBT drivers |

**Transfer function:** 1A output = 0.4V setpoint input (gain = 2.5 A/V)

### 2.6 Standard Operating Procedure
1. **Diagnostics:** T-adapter oscilloscope taps on Measure and Setpoint lines.
2. **Power-up:** energize 12V, 5V, ±15V first; then the external PID controller.
3. **PID config:** clamp output to 0–10V *before* energizing main power. Known-good baseline: **P = 0.8, I = 2.6×10⁵, D = 0, Offset = 4.5.**
4. **Active operation:** fans on → energize Main Power → apply setpoint.
5. **Shutdown (in order):** PID controller off → Main Power off → all logic/peripheral power off. Never hot-plug the coil, power, or logic while energized — risk of massive inductive spikes or floating-gate destruction.

### 2.7 Protection Circuitry (as designed)
- **Flyback diode** across the coil + transducer series combination
- **Varistor (MOV)** in parallel with the flyback diode, clamping transients beyond the diode's reverse rating
- **Ground-clamping diode** on the transducer secondary output, preventing lethal negative excursions

### 2.8 Measured Performance (Phase 2 report, July 2026)
- Rise time: 26µs, Fall time: 30µs
- Matches theory: MOSFET gate charge Qg ≈ 610nC ÷ PID's 20mA max output = **30.5µs theoretical minimum** — confirms the PID's unbuffered drive into the MOSFET gate is the bottleneck, not the coil or control loop.
- Thesis target was 500µs @ 1–2A — prototype greatly exceeded this, but partly because no coil was connected during this test.

### 2.9 As-Built Modification
- **X-Bias axis IGBT gate driver (EVAL-ADUM4221-1EBZ) failed** — IGBT stuck open (OFF).
- **Fix applied:** physical bypass jumper soldered from MOSFET Source directly to IGBT Emitter, removing the IGBT from the current path entirely. X-Bias operates normally without its static switch.

### 2.10 Original BOM
| Component | Part Number | Role |
|---|---|---|
| Linear MOSFET | IXTK60N50L2 | 500V/60A, main VCCS control element |
| IGBT | AUIRGPS4070D0 | 400V/70A, static series switch (bypassed on X-Bias) |
| IGBT Driver | EVAL-ADUM4221-1EBZ | Isolated half-bridge gate driver |
| Current Transducer | LEM ITN 12-P ULTRASLAB | 12.5A, 1:250 ratio, closed-loop |
| Burden Resistor | 100Ω precision | Converts secondary current to 0.4V/A |
| Ref. Resistors | 3.3kΩ | Reference voltage dividers |
| TVS Diode | 5KP58A | 5000W transient suppressor |
| Varistor | V150LA20AP | 150V RMS MOV |

---

## 3. Phase 3 — Recovery & Debug Log

Key findings during recovery of the stored prototype, in roughly chronological order:

- **Y-axis heating incident:** driving a 3.3kΩ "coil" stand-in with a 770ms/0–272mV square wave (calculated ~20mA) produced smoke after 20s. Root cause not fully resolved in the log; connections re-checked, Y subsequently confirmed working (16ms response @ 1V, P=0.7).
- **Baseline good-axis behavior (Y, Z, MOT):** working correctly at setpoint 1kHz, 405mV high / 0V low, 50% duty.
- **X-axis fault confirmed:** IGBT driver draws 0.47A and behaves like a short circuit — consistent with/independent confirmation of the Phase 2 as-built failure.
- **Response time re-verification:** 400mV → 1A, 800mV → 2A (matches 2.5A/V transfer function); ~30µs rise time across axes, matching the theoretical unbuffered-gate limit — confirmed even with a coil connected (previously suspected the fast time was a no-load artifact; it wasn't).
- **5.5A step-response replication:** square wave modulation at 3kHz and 5kHz — visible phase lag and oscillation approaching the RC limit of the MOSFET gate, confirming the analog PID cannot drive the gate capacitance fast enough without a buffer. Slight overshoot (~10%) noted on square-wave transitions, consistent with the original thesis's reported ~10% overshoot on current drop.

### 3.1 Redesign Directions Explored (superseded by Phase 4, kept for reference)
- **Add a gate buffer (BUF634A or LT1210)** between the PID output and MOSFET gate to eliminate the 30µs charge-time bottleneck (theoretical improvement to <2.4µs with a 200mA-class buffer).
- **Buffer requires shifting to a low-side MOSFET topology** — a high-side MOSFET has a floating source (rides on Vout), which a ground-referenced buffer can't drive directly.
- **Open question at the time:** the flyback diode (D1) in the original schematic didn't match a standard flyback topology and had no clear place in a low-side redesign — flagged to the thesis author (Mithilesh), no reply received in the log.
- **IGBT necessity questioned:** no experimental procedure in the log ever toggled the IGBT as an active switch; PWM/DISABLE logic pins appeared hardwired via physical jumpers regardless of commanded logic state. Removing the IGBT was estimated to save ~0.6W/A of wasted conduction loss with no downside, since shutdown speed is limited by the varistor's dissipation rate, not the IGBT.
- **New H-bridge candidates evaluated:**
  - Pololu G2 High-Power Motor Driver 24v21
  - Commercial **Intelligent Power Module (IPM)** — IM06B20AC1, using 2 of its 3 phases as an H-bridge (5 samples ordered; needs custom mounting PCB)
- **LT1210 buffer reference design** (for the low-side MOSFET gate, if this path is revisited):
  - Non-inverting gain-of-1 buffer: PID signal → +IN; RF = 750Ω (1%, 1/4W metal film — not a wire short) between OUT and −IN
  - CCOMP = 10nF between OUT and COMP (required for stability into the MOSFET's gate capacitance)
  - RPD = 10kΩ gate pull-down (prevents floating gate in shutdown)
  - CBYP = 100nF + CBULK = 4.7µF on both V+ and V− pins, placed close to the package
  - 10Ω gate resistor between buffer output and MOSFET gate

**Status:** This MOSFET/IGBT/H-bridge line of development was superseded by the Phase 4 linear amplifier redesign below, which takes a different approach (discrete BJT push-pull instead of a single linear MOSFET pass element).

---

## 4. Phase 4 — New Linear VCVS Amplifier Redesign (current work)

### 4.1 Topology
A **voltage-controlled voltage source (VCVS)** built around a precision high-voltage op-amp (OPA455) providing gain/accuracy, buffered by a **two-stage complementary emitter-follower (EF2)** discrete BJT output stage for current capacity — a different architectural approach from the Phase 1–3 single-MOSFET linear VCCS.

**Signal path:** `COMMAND → OPA455 (+IN) → HV_DRIVE_NODE (op-amp output) → predrive/bias network (Q12 VBE multiplier) → Q13/Q14 driver pair (first EF stage) → 5× paralleled complementary output pairs Q2–Q11 (second EF stage) → COIL_DRIVE`

### 4.2 Key Stages
- **U2 (OPA455IDDAR):** high-voltage precision op-amp, powered from local ±30V_OP rails (4.7µF + 100nF local decoupling)
- **Q12 (BD139) VBE multiplier ("bias spreader"):** sets Class-AB quiescent bias between NPN_PREDRIVE and PNP_PREDRIVE. Base current sourced via R2 (10K) from HV_DRIVE_NODE; R23 (2.2K, fixed) + RV2 (trim pot) form the divider. Trim range covers roughly 2.0–3.3V bias spread for RV2 ≈ 0–2K, bracketing the ~2.4–2.9V needed across the full 4-junction quiescent loop.
- **Q13/Q14 (MJE15032/MJE15033):** first-stage complementary driver pair, emitters form NPN_DRIVE_BUS / PNP_DRIVE_BUS
- **Q2–Q11 (5× MJL21194 NPN / MJL21193G PNP pairs):** paralleled output stage, each with 2.2Ω base resistor + 0.22Ω emitter ballast resistor for current sharing, driving common **COIL_DRIVE** node
- **D1/D2 flyback diodes:** COIL_DRIVE clamped to +30V/−30V rails, backed by **1000µF bulk caps** on each rail — provides fast forced-discharge path for coil energy on shutdown (~290µs to zero from 10A, vs. 1.35ms natural L/R decay)
- **R1 (100Ω):** bridges NPN_DRIVE_BUS to PNP_DRIVE_BUS, a driver-stage minimum-current bleed (~12–14mA) keeping Q13/Q14 out of a dead zone near crossover; independent of and non-conflicting with the protection diodes below

### 4.3 Reverse VEBO Protection (added during this redesign)
**Problem identified:** MJL21193/94, MJE15032/33, and BD139 all share a **VEBO of only 5V**, despite high VCEO/VCBO ratings (250–400V). During frequent power-down cycling, the small local op-amp supply caps (4.7µF) collapse much faster than the main 1000µF bulk rail caps — this asymmetry can force >5V reverse bias across base-emitter junctions, causing cumulative hFE degradation over repeated power cycles.

**Fix:** 5 clamp diodes added, each pinning the vulnerable junction's reverse voltage to a single forward diode drop instead of letting it approach the 5V breakdown:

| Diode | Protects | Type | Part | Anode | Cathode |
|---|---|---|---|---|---|
| D6 | Q13 (NPN driver) | NPN | 1N4148W | NPN_DRIVE_BUS (emitter) | NPN_PREDRIVE (base) |
| D7 | Q14 (PNP driver) | PNP | 1N4148W | PNP_PREDRIVE (base) | PNP_DRIVE_BUS (emitter) |
| D3 | NPN output bank (Q2/6/8/10 etc.) | NPN | SS34 (Schottky) | COIL_DRIVE (emitter) | NPN_DRIVE_BUS (base) |
| D4 | PNP output bank | PNP | SS34 (Schottky) | PNP_DRIVE_BUS (base) | COIL_DRIVE (emitter) |
| D5 | Q12 (bias spreader) | NPN | 1N4148W | PNP_PREDRIVE (emitter) | NPN_PREDRIVE (collector) |

Rule used throughout: NPN → clamp emitter-exceeds-base; PNP → clamp base-exceeds-emitter. Because all 5 output devices per bank share the same base bus and same emitter bus, **one diode per bank protects all 5 devices simultaneously** — no need for 10 individual diodes. D5 clamps Q12's collector-emitter span rather than reaching its buried base-tap node, which works because the base always sits resistively between collector and emitter.

**Placement:** D6/D2/D5 placed immediately at their respective transistor pads (D5 piggybacked on C22's existing predrive-to-predrive pads for minimum trace length); D3/D4 placed centrally within their respective 5-device row to minimize worst-case trace inductance to any single device.

**Open item:** confirm actual diode polarity in-tool (anode/cathode pin assignment, not silkscreen glyph) before further fabrication — logical analysis showed 4 of 5 diodes require "anode on bottom" while only D6 requires "anode on top," so if all five were placed with identical symbol rotation, at most one can be correct as drawn.

**Also flagged (unresolved):** thermal coupling of Q12 to the output-device heatsink for proper bias tracking under sustained 10A operation — not yet confirmed against physical layout.

### 4.4 Board Bring-Up — Open Issue
**Symptom:** With the board's six power/ground connections all made (no coil, no signal connected), PSU current settles at ~5A. Disconnecting *any single one* of the six restores normal (near-zero) current.

**Leading hypothesis: ground loop from sharing one bipolar PSU for both signal ground (SGND) and power ground (PGND).** The board deliberately separates SGND (op-amp reference) from PGND (main power return) with a single intended tie point; wiring both grounds back to the same PSU terminal creates a second, low-resistance tie point externally, closing a loop that a small SGND–PGND potential difference can drive several amps through — consistent with the "only fails when the loop is fully closed" symptom.

**Alternative hypothesis:** Class-AB bias (RV2) set too high, causing real quiescent current through the output stage — but this requires all six connections to complete the bias loop across both the signal-side and power-side supplies, which also fits the symptom.

**Diagnostic recommended, not yet performed:**
1. Continuity/resistance check between SGND and PGND at the board connectors, PSU disconnected — determines whether they're already tied on-board.
2. Voltage-drop measurement across an emitter ballast resistor (e.g. R4/R5, 0.22Ω) with the 5A condition present — distinguishes "current flowing through the amplifier's active devices" (bias runaway) from "current bypassing the signal path entirely" (ground loop).

**Status: unresolved, pending the above measurements.**

---

## 5. Consolidated Open Items

- [ ] Diagnose the 5A idle-current symptom (ground loop vs. bias runaway) — see §4.4
- [ ] Confirm all 5 protection diode polarities in-CAD against the table in §4.3 before further board spins
- [ ] Confirm thermal coupling of Q12 to the output-device heatsink
- [ ] Confirm RV2's actual pot value against the trim-range calculation in §4.2
- [ ] Verify D1/D2 flyback diode current/energy rating against repetitive 44mJ shutdown pulses (L ≈ 0.886mH, 10A, τ ≈ 1.35ms coil time constant)
- [ ] Decide whether any Phase 3 low-side MOSFET / IPM H-bridge work is still relevant, or fully superseded by the Phase 4 linear amplifier approach

---

## 6. Reference Calculations

- **Coil parameters (as measured):** τ = L/R_dc = 1.35ms, R_dc = 0.656Ω → **L ≈ 0.886mH**
- **Stored energy at 10A:** E = ½LI² ≈ **44mJ** per full-current shutdown event
- **Forced discharge time (via D1/D2 + 1000µF rail caps):** dI/dt ≈ V_clamp/L ≈ 34,700 A/s → **≈290µs to zero from 10A**
- **VBE multiplier (Q12) trim math:** M = 1 + R2/(R23+RV2) = 1 + 10000/(2200+RV2); target M ≈ 4–4.8 for a ~2.4–2.9V quiescent spread across the 4-junction loop
