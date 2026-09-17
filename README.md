# Precision Bias Coil Current Driver — Project README

## 🔑 TL;DR

**Precision ±10–12.5A analog current driver for magnetic bias coils in a cold-atom/MOT physics experiment, sub-100µs response.**

- 🔧 **Diagnosed & recovered** a dead legacy MOSFET/IGBT current-source prototype — traced a failed IGBT gate driver, resolved ground-loop and thermal issues
- 🔁 **Redesigned from scratch**: precision op-amp (OPA455) + discrete BJT push-pull output stage, replacing the old MOSFET/IGBT topology
- 🛡️ **Added a 5-diode protection network** to prevent transistor degradation from reverse-bias stress during power cycling
- ✅ **Ground loop on new board — resolved**: switched from a shared bipolar PSU to individual PSUs per rail
- 🔴 **Root cause found for 2nd bring-up fault**: −30V rail was pinned to max current regardless of tuning — traced to **D5 and D6 wired backwards** (design-file-level diode orientation error, not per-unit damage). Final confirmation (direct voltage measurement + physical inspection) still pending.
- 🚧 **Status:** root cause identified, rework/re-verification in progress

---

![https://github.com/DavodChuweiLiao/10-BJT-VCCS-Diagnosis/blob/main/Layout.png?raw=true](https://github.com/DavodChuweiLiao/10-BJT-VCCS-Diagnosis/blob/main/Layout.png?raw=true)
![https://github.com/DavodChuweiLiao/10-BJT-VCCS-Diagnosis/blob/main/Screenshot%202026-09-14%20211057.png?raw=true](https://github.com/DavodChuweiLiao/10-BJT-VCCS-Diagnosis/blob/main/Screenshot%202026-09-14%20211057.png?raw=true)

![https://github.com/DavodChuweiLiao/10-BJT-VCCS-Diagnosis/blob/main/Screenshot%202026-09-16%20231345.png?raw=true](https://github.com/DavodChuweiLiao/10-BJT-VCCS-Diagnosis/blob/main/Screenshot%202026-09-16%20231345.png?raw=true)

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

| Diode | Protects | Type | Part | Required Anode | Required Cathode | **As-built (verified from footprint, 2nd board)** |
|---|---|---|---|---|---|---|
| D6 | Q13 (NPN driver) | NPN | 1N4148W | NPN_DRIVE_BUS (emitter) | NPN_PREDRIVE (base) | ❌ **BACKWARDS** — built anode=HV_DRIVE_NODE/NPN_PREDRIVE, cathode=NPN_DRIVE_BUS |
| D7 | Q14 (PNP driver) | PNP | 1N4148W | PNP_PREDRIVE (base) | PNP_DRIVE_BUS (emitter) | ✅ Correct |
| D3 | NPN output bank (Q2/6/8/10 etc.) | NPN | SS34 (Schottky) | COIL_DRIVE (emitter) | NPN_DRIVE_BUS (base) | ✅ Correct |
| D4 | PNP output bank | PNP | SS34 (Schottky) | PNP_DRIVE_BUS (base) | COIL_DRIVE (emitter) | ✅ Correct |
| D5 | Q12 (bias spreader) | NPN | 1N4148W | PNP_PREDRIVE (emitter) | NPN_PREDRIVE (collector) | ❌ **BACKWARDS** — built anode=HV_DRIVE_NODE/NPN_PREDRIVE, cathode=PNP_PREDRIVE |

Rule used throughout: NPN → clamp emitter-exceeds-base; PNP → clamp base-exceeds-emitter. Because all 5 output devices per bank share the same base bus and same emitter bus, **one diode per bank protects all 5 devices simultaneously** — no need for 10 individual diodes. D5 clamps Q12's collector-emitter span rather than reaching its buried base-tap node, which works because the base always sits resistively between collector and emitter.

**Placement:** D6/D2/D5 placed immediately at their respective transistor pads (D5 piggybacked on C22's existing predrive-to-predrive pads for minimum trace length); D3/D4 placed centrally within their respective 5-device row to minimize worst-case trace inductance to any single device.

**🔴 FINDING (confirmed via footprint/schematic cross-check on 2nd board's PCB files): D5 and D6 are wired backwards.** This is a design-file-level error (footprint/schematic diode orientation), not an assembly or component defect — it reproduces identically on every board built from the same source files, which is exactly what was observed (issue persisted on a brand-new board with independently verified-good components).

**Why D5 backwards is the dominant fault:** D5 sits directly across NPN_PREDRIVE–PNP_PREDRIVE, the two nodes Q12's VBE multiplier is supposed to hold ~2.4–2.9V apart. Built backwards, its forward-conduction condition (anode > cathode) is now `NPN_PREDRIVE > PNP_PREDRIVE` — which is the *normal, designed-in relationship, true 100% of the time*, not a rare fault condition. So instead of sitting silently reverse-biased (as intended) and only engaging during an abnormal event, D5 conducts continuously from power-up, clamping the entire quiescent bias spread down to ~0.5–0.6V (one diode drop) regardless of where RV2 is set. This fully explains the bring-up symptom in §4.4b below. D6 backwards is a real error too (partially shunts Q13's own B-E path in normal operation) but is secondary to D5's effect.

**Verification status:** confirmed at the footprint/schematic level (pad-to-net mapping cross-checked against required polarity table above). **Not yet confirmed by direct in-circuit voltage measurement** (expect ~0.5–0.6V across D5 if the diagnosis is correct, vs. ~2.4–2.9V if healthy) or by physical cathode-band inspection against board silkscreen (would additionally rule out an assembly-house placement error independent of the design file itself).

**Fix path:** correct D5/D6 orientation in the schematic/footprint library association before the next board spin (this is a source-file fix, not per-unit rework); for the boards in hand, rework by desolder-and-flip or a dead-bug bodge diode in the correct orientation for validation.

**Also flagged (unresolved):** thermal coupling of Q12 to the output-device heatsink for proper bias tracking under sustained 10A operation — not yet confirmed against physical layout.

### 4.4 Board Bring-Up — Issue Log

#### 4.4a Ground loop (RESOLVED)
**Symptom:** With the board's six power/ground connections all made (no coil, no signal connected), PSU current settled at ~5A. Disconnecting *any single one* of the six restored normal (near-zero) current.

**Root cause: ground loop from sharing one bipolar PSU for both signal ground (SGND) and power ground (PGND).** The board deliberately separates SGND (op-amp reference) from PGND (main power return) with a single intended tie point; wiring both grounds back to the same PSU terminal created a second, low-resistance tie point externally, closing a loop that a small SGND–PGND potential difference could drive several amps through.

**Fix:** switched to individual/isolated PSUs per rail instead of one shared bipolar supply. **Confirmed resolved** — no coil path connected, no runaway current from the ground loop.

#### 4.4b PNP rail pinned to max current, unresponsive to tuning (ROOT CAUSE IDENTIFIED, pending final confirmation)
**Symptom:** After resolving 4.4a, a second, distinct fault emerged: the −30V (PNP) side of the output stage always pulls PSU current limit (10A), regardless of RV2 trim position or input command voltage. With +30V off, the −30V side still pulls max current. With only +30V on (−30V off), no current flows at all. **Reproduced identically on a second, independently assembled board** with individually verified-good components (no shorted transistors, healthy flyback diodes, correctly-functioning RV2 pot).

**Diagnostic path (in order, ruling out each candidate):**
1. Q12 (BD139) — diode-tested, no shorts found. Cleared.
2. D1/D2 flyback diodes — D2 isolated and tested: healthy (open one direction, ~0.3V forward the other). D1 not yet isolated-tested but in-circuit reading was consistent with D2's, not separately suspicious.
3. Paralleled PNP output transistors (Q2/Q5/Q7/Q9/Q11) — pinout confirmed against board nets (base→2.2Ω resistor→drive bus; collector→direct to rail; emitter→0.22Ω resistor→COIL_DRIVE), consistent with schematic. In-circuit C-E readings that initially looked suspicious (~0.06V) were later understood as a side-effect of the D1/D2 in-circuit measurement ambiguity, not independent transistor faults.
4. RV2 (trim pot) — pin 2–3 short confirmed to be an intentional wiper-to-end tie (standard trim-pot wear protection technique), not a fault; full 0.6–3kΩ sweep confirmed on pin 1–3 while rotating. **RV2 cleared.**
5. **D5 and D6 (predrive protection diodes) — confirmed wired backwards** via footprint/schematic net cross-check (see §4.3). **This is the identified root cause.**

**Mechanism:** D5, built backwards, continuously clamps the NPN_PREDRIVE–PNP_PREDRIVE bias spread to ~0.5–0.6V instead of allowing Q12/RV2 to set the designed ~2.4–2.9V — starving the entire 4-junction bias loop regardless of RV2 position, and consistent with reproducing identically on a second board (a design-file-level fault, not a per-unit component issue).

**Remaining verification before calling this fully closed:**
- [ ] Direct in-circuit voltage measurement across D5 (NPN_PREDRIVE to PNP_PREDRIVE) — expect ~0.5–0.6V if diagnosis is correct, ~2.4–2.9V if not
- [ ] Physical cathode-band inspection of D5/D6 against board silkscreen (rules out an assembly-house placement error independent of the design file)
- [ ] Rework D5/D6 to correct orientation and re-test full bias behavior (RV2 responsiveness, both rails balanced)

---

## 5. Consolidated Open Items

**Resolved this session:**
- [x] ~~Diagnose the 5A idle-current ground-loop symptom~~ — resolved by switching to individual PSUs per rail (§4.4a)
- [x] ~~Confirm all 5 protection diode polarities~~ — done: D3, D4, D7 correct; **D5 and D6 confirmed wired backwards** (§4.3, §4.4b)
- [x] ~~Verify RV2 pot behavior~~ — confirmed working correctly (intentional wiper-to-end tie, full sweep range)

**Still open:**
- [ ] Direct in-circuit voltage confirmation across D5 (expect ~0.5–0.6V) — final confirmation of the backwards-diode diagnosis
- [ ] Physical cathode-band inspection of D5/D6 vs. board silkscreen — rules out assembly error vs. design-file error
- [ ] Rework D5/D6 on existing board(s) to correct orientation; re-test full bias behavior after fix
- [ ] Correct D5/D6 orientation in schematic/footprint source files before any further board spins
- [ ] Confirm thermal coupling of Q12 to the output-device heatsink
- [ ] Verify D1/D2 flyback diode current/energy rating against repetitive 44mJ shutdown pulses (L ≈ 0.886mH, 10A, τ ≈ 1.35ms coil time constant)
- [ ] Isolated (out-of-circuit) test of D1, to fully match the confirmation already done on D2
- [ ] Decide whether any Phase 3 low-side MOSFET / IPM H-bridge work is still relevant, or fully superseded by the Phase 4 linear amplifier approach

---

## 6. Reference Calculations

- **Coil parameters (as measured):** τ = L/R_dc = 1.35ms, R_dc = 0.656Ω → **L ≈ 0.886mH**
- **Stored energy at 10A:** E = ½LI² ≈ **44mJ** per full-current shutdown event
- **Forced discharge time (via D1/D2 + 1000µF rail caps):** dI/dt ≈ V_clamp/L ≈ 34,700 A/s → **≈290µs to zero from 10A**
- **VBE multiplier (Q12) trim math:** M = 1 + R2/(R23+RV2) = 1 + 10000/(2200+RV2); target M ≈ 4–4.8 for a ~2.4–2.9V quiescent spread across the 4-junction loop
