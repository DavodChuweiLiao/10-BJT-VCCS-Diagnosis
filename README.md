# Precision Bias Coil Current Driver — Project README

## 🔑 TL;DR

**Precision ±10–12.5A analog current driver for magnetic bias coils in a cold-atom/MOT physics experiment, sub-100µs response.**

- 🔧 **Diagnosed & recovered** a dead legacy MOSFET/IGBT current-source prototype — traced a failed IGBT gate driver, resolved ground-loop and thermal issues
- 🔁 **Redesigned from scratch**: precision op-amp (OPA455) + discrete BJT push-pull output stage, replacing the old MOSFET/IGBT topology
- 🛡️ **Added a 5-diode protection network** to prevent transistor degradation from reverse-bias stress during power cycling
- ✅ **Fault #1 SOLVED — floating OPA455 E/D pin.** Left unconnected per datasheet's "self-enables" note, it measured only **0.06V** above E/D Com — inside the disable window. The output stage sat at 160kΩ, dragging the predrive network to the negative rail and turning the whole PNP output bank hard on. Fixed with a 10k/4.7k divider holding E/D at ~4.8V; op-amp now tracks correctly (+IN 4V → OUT 4V).
- 🔴 **Fault #2 OPEN — SGND and PGND are not connected.** Measured **200kΩ** between them; the design has no intentional single-point tie. Op-amp section has no defined reference vs. the power section → negative rail still collapses to ~1.4V with 3 of 5 PNP devices conducting. Next test: manually bridge the grounds at the PSU.
- 🚧 **Status:** running at ±15V for bring-up; one fault fixed, one identified and awaiting test

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
| New amplifier redesign | current | OPA455-based op-amp + discrete complementary BJT push-pull (EF2) output stage, replacing the MOSFET/IGBT linear-VCCS approach; PCB fabricated, 2 boards built, under bring-up |

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
- **U2 (OPA455IDDAR):** high-voltage precision op-amp, powered from local ±30V_OP rails (4.7µF + 100nF local decoupling). Pin 2 (−IN) and pin 6 (OUT) are the same net (HV_DRIVE_NODE) → **unity-gain follower with the BJT output stage outside the feedback loop** (see §4.5 for implications). PowerPAD (pin 9) tied to V− per datasheet requirement.
- **Q12 (BD139) VBE multiplier ("bias spreader"):** sets Class-AB quiescent bias between NPN_PREDRIVE and PNP_PREDRIVE. Base current sourced via R2 (10K) from HV_DRIVE_NODE; R23 (2.2K, fixed) + RV2 (trim pot) form the divider. Trim range covers roughly 2.0–3.3V bias spread for RV2 ≈ 0–2K, bracketing the ~2.4–2.9V needed across the full 4-junction quiescent loop.
- **Q13/Q14 (MJE15032/MJE15033):** first-stage complementary driver pair, emitters form NPN_DRIVE_BUS / PNP_DRIVE_BUS
- **Q2–Q11 (5× MJL21194 NPN / MJL21193G PNP pairs):** paralleled output stage, each with 2.2Ω base resistor + 0.22Ω emitter ballast resistor for current sharing, driving common **COIL_DRIVE** node
- **D1/D2 flyback diodes:** COIL_DRIVE clamped to +30V/−30V rails, backed by **1000µF bulk caps** on each rail — provides fast forced-discharge path for coil energy on shutdown (~290µs to zero from 10A, vs. 1.35ms natural L/R decay)
- **R1 (100Ω):** bridges NPN_DRIVE_BUS to PNP_DRIVE_BUS, a driver-stage minimum-current bleed (~12–14mA) keeping Q13/Q14 out of a dead zone near crossover

### 4.3 Reverse VEBO Protection
**Problem identified:** MJL21193/94, MJE15032/33, and BD139 all share a **VEBO of only 5V**, despite high VCEO/VCBO ratings (250–400V). During frequent power-down cycling, the small local op-amp supply caps (4.7µF) collapse much faster than the main 1000µF bulk rail caps — this asymmetry can force >5V reverse bias across base-emitter junctions, causing cumulative hFE degradation over repeated power cycles.

**Fix:** 5 clamp diodes, each pinning the vulnerable junction's reverse voltage to a single forward diode drop instead of letting it approach the 5V breakdown:

| Diode | Protects | Type | Part | Required Anode | Required Cathode | As-built (physical inspection) |
|---|---|---|---|---|---|---|
| D6 | Q13 (NPN driver) | NPN | 1N4148W | NPN_DRIVE_BUS (emitter) | NPN_PREDRIVE (base) | ✅ Correct |
| D7 | Q14 (PNP driver) | PNP | 1N4148W | PNP_PREDRIVE (base) | PNP_DRIVE_BUS (emitter) | ✅ Correct |
| D3 | NPN output bank | NPN | SS34 (Schottky) | COIL_DRIVE (emitter) | NPN_DRIVE_BUS (base) | ✅ Correct |
| D4 | PNP output bank | PNP | SS34 (Schottky) | PNP_DRIVE_BUS (base) | COIL_DRIVE (emitter) | ✅ Correct |
| D5 | Q12 (bias spreader) | NPN | 1N4148W | PNP_PREDRIVE (emitter) | NPN_PREDRIVE (collector) | ✅ Correct |

Rule used throughout: NPN → clamp emitter-exceeds-base; PNP → clamp base-exceeds-emitter. Because all 5 output devices per bank share the same base bus and same emitter bus, **one diode per bank protects all 5 devices simultaneously** — no need for 10 individual diodes. D5 clamps Q12's collector-emitter span rather than reaching its buried base-tap node, which works because the base always sits resistively between collector and emitter.

**Placement:** D6/D7/D5 at their respective transistor pads (D5 piggybacked on C22's predrive-to-predrive pads for minimum trace length); D3/D4 centrally within their respective 5-device row to minimize worst-case trace inductance.

> ⚠️ **RETRACTED FINDING — kept as a methodology note.** An earlier analysis, based on reading the diode triangle glyphs in KiCad footprint screenshots, concluded D5 and D6 were wired backwards and blamed them for the bring-up fault. **This was wrong.** Direct physical inspection of the cathode bands on the populated boards confirmed all five diodes are correctly oriented. Lesson: a pixel-level read of a CAD symbol is not evidence — verify polarity against the physical part's cathode band, or better, against a direct electrical measurement. The 0.7V that appeared to "confirm" the backwards-D5 theory turned out to be an unrelated consequence of the real fault (§4.4b).

**Also flagged (unresolved):** thermal coupling of Q12 to the output-device heatsink for proper bias tracking under sustained 10A operation — not yet confirmed against physical layout.

### 4.4 Board Bring-Up — Issue Log

#### 4.4a Initial 5A idle current (RESOLVED — misdiagnosed at the time)
**Symptom:** With all six power/ground connections made (no coil, no signal), PSU current settled at ~5A. Disconnecting any single one of the six restored near-zero current.

**Original hypothesis (wrong):** a ground loop from sharing one bipolar PSU across SGND and PGND.

**Actual cause (established later):** the OPA455's output stage was disabled (§4.4b). With one shared bipolar PSU, SGND and PGND *were* tied through the supply common, so the disabled op-amp's collapsed output dragged the predrive network to the negative rail and turned the PNP output bank hard on — drawing 5A. Unplugging any lead broke that path, which mimicked ground-loop behavior. Switching to individual PSUs per rail removed the ground tie entirely, which stopped the 5A but introduced the floating-ground problem now tracked as §4.4c.

#### 4.4b Op-amp output stage disabled — floating E/D pin (✅ ROOT CAUSE CONFIRMED AND FIXED)
**Symptom:** The negative rail always pulls PSU current limit regardless of RV2 position or input command. With +30V off, −30V still maxes out. With only +30V on, nothing happens. **Reproduced identically on a second, independently assembled board** with individually verified-good components.

**Diagnostic path (each candidate ruled out in turn):**
1. **Q12 (BD139)** — diode-tested, no shorts. Cleared.
2. **D1/D2 flyback diodes** — D2 isolated and tested healthy (open one way, ~0.3V forward the other). Cleared.
3. **PNP output transistors (Q2/Q5/Q7/Q9/Q11)** — pinout confirmed against board nets (pad 1 base → 2.2Ω → drive bus; pad 2 collector → rail; pad 3 emitter → 0.22Ω → COIL_DRIVE). In-circuit C-E readings that looked suspicious (~0.06V) were an artifact of parallel paths through D1/D2, not device faults. Cleared.
4. **RV2 (trim pot)** — the pin 2–3 "short" is an intentional wiper-to-end tie (standard trim-pot wear protection); full 0.6Ω–3kΩ sweep confirmed on pin 1–3. Cleared.
5. **D5/D6 orientation** — investigated and **retracted** (see §4.3 note). Cleared.
6. **OPA455 E/D (enable/disable) pin — CONFIRMED ROOT CAUSE.**

**The schematic leaves E_D (pin 8) unconnected, with E_D_Com (pin 1) tied to SGND.** TI's datasheet §7.3.4 says a floating E/D pin self-enables — but only via an internal **1µA** source holding it ~2V above E/D Com, against a disable threshold only 0.8V away. TI explicitly warns this is fragile and easily overpowered.

**Measured: E/D sat at just 0.06V above E/D Com** — squarely inside the disable window (spec: disabled = E/D Com to E/D Com + 0.35V).

**Confirming signature:** with +IN commanded to 4V, both −IN and OUT read **2.5V** — attenuated but still tracking. This is the textbook disabled-output behavior from §7.3.4: output impedance rises to ~160kΩ while the inputs stay active, so the signal feeds through the 160kΩ against the external network (R2 + R23/RV2 + R24 ≈ 13.5kΩ) and appears reduced. A dead amp would give a rail or nothing; an enabled follower would give a clean 4V.

**This explains every downstream symptom:** op-amp output collapses toward the negative rail → predrive nodes collapse with it → Q14 and the PNP output bank driven hard on → negative rail pinned at current limit → RV2 irrelevant (the bias spreader has no working operating point) → reproduces on every board (it's a schematic-level choice, not damage).

**Fix applied (bench bodge):** resistive divider from +15V_OP → **10kΩ** → E/D (pin 8) → **4.7kΩ** → SGND, giving **~4.8V** on E/D — mid-window (enable = 2.5–5V above E/D Com), under the 7V E/D-to-E/D-Com absolute max, and ~1mA of pull-up (≈20× the pin's ~50µA draw) so leakage can't drag it down.

**Result: ✅ CONFIRMED.** With the divider fitted and only the signal-side rails powered: **+IN = 4V, −IN = 4V, OUT (HV_DRIVE_NODE) = 4V.** The follower closes its loop correctly.

#### 4.4c SGND and PGND are not connected (🔴 OPEN — next test)
**Symptom:** With the op-amp now working, the negative main rail still collapses. At a 0.17A limit it sits at **−1.4V**; raising the limit to **4A for 5 seconds did not bring the rail up** (still ~−1.2V).

**Measurements taken:**
| Measurement | Value | Interpretation |
|---|---|---|
| −15V rail at board, under limit | −1.2 to −1.4V | Supply is in constant-current mode, output nearly collapsed |
| Rail voltage vs. current (0.17A → 4A) | ~1.4V → ~1.2V | **Nearly constant across a 20× current range** → forward-biased junctions clamping, *not* a resistive short |
| Supply terminal vs. board pad | no significant drop | Rules out cable/connector resistance — the load is on the board |
| PNP_DRIVE_BUS → −15V (unpowered) | 0.5 MΩ | No short |
| COIL_DRIVE → +15V (unpowered) | 800 kΩ | Normal |
| COIL_DRIVE → −15V (unpowered) | 80 kΩ | Asymmetric vs. +15V side, but far too high to draw 0.2A — benign junction/D2 leakage |
| Voltage across R1 (100Ω) | 0.389V → 3.9mA | **No driver shoot-through** — Q13/Q14 are not the current path |
| Ballast resistors, PNP side (0.22Ω) | **3 of 5 at 24mV** (≈109mA each, ~327mA total) | **The PNP output bank is the load** |
| Ballast resistors, NPN side | all 0V | NPN bank fully off — one-sided conduction, not shoot-through |
| RV2 position during all of the above | max resistance = **minimum** bias spread | The pot has no authority over this fault |
| **SGND to PGND resistance** | **200 kΩ** | 🔴 **The two grounds are not actually tied** |

**Root cause (identified, test pending):** the design separates SGND (op-amp reference) from PGND (power return) but has **no intentional single-point tie** — the 200kΩ is just semiconductor leakage. The op-amp section therefore has no defined potential relative to the power section, so the drive delivered to Q13/Q14 and the output bank is arbitrary. It has settled somewhere that forward-biases the PNP bank, which matches the measured 3-of-5 PNP conduction with the NPN bank completely off.

Two further details consistent with this reading:
- The 3-of-5 split (rather than all five) is expected near-crossover behavior: at 24mV the 0.22Ω ballasts provide almost no degeneration, so the three devices with the lowest Vbe hog the current. Not a defect in its own right.
- RV2 was at minimum-bias throughout, so "bias set too high" is excluded as an explanation.

**Next test:** bridge SGND to PGND with a single wire at the PSU common (one point only — a second tie would create the genuine ground loop wrongly diagnosed in §4.4a), then bring up at 0.2A limit with +IN = 0V and check:
- [ ] Does the negative rail now reach −15V instead of clamping at 1.4V?
- [ ] Do the PNP ballast resistors drop to ~0V?
- [ ] Does NPN_PREDRIVE → PNP_PREDRIVE land in the 2.4–2.9V range and respond to RV2?

**Design fix for next spin:** add a deliberate single-point SGND–PGND bridge (net tie, 0Ω link, or ferrite). A split-ground design needs exactly one; this board has none.

### 4.5 Latent Issues Found During Bring-Up (not yet causing failures)

- **🔴 C22 (1µF) violates the OPA455 capacitive load spec by ~5000×.** C22 sits across the predrive nodes, connected to the op-amp output through only R26 = 10Ω. The datasheet rates `CLOAD Capacitive load drive` at **200pF**. This is a genuine stability risk that was completely masked while the output stage was disabled — it only becomes visible now that the amp actually drives. **Action:** scope HV_DRIVE_NODE for oscillation as soon as the rails come up; if present, add a proper isolation resistor between OUT and the predrive node, or relocate C22. A DMM in AC mode is *not* sufficient to detect this (oscillation will be in the 100kHz–MHz range, beyond most handheld AC-volts bandwidth).
- **Status Flag (pin 5) is left unconnected**, so overtemperature and overcurrent faults are invisible. Both also produce a high-impedance output — the same signature as E/D shutdown. **Action:** add a 10kΩ pull-up to 5V (referenced to E/D Com) and a test point; low = active fault.
- **Feedback is taken at the op-amp output, not after the output stage.** Pin 2 (−IN) and pin 6 (OUT) share the HV_DRIVE_NODE net, making this a unity-gain follower with the entire EF2 BJT stage *outside* the loop. Consequence: the output transistors' Vbe drops and crossover distortion are **not corrected by feedback**, so COIL_DRIVE will not accurately track COMMAND. For a precision current driver this is a real limitation — consider moving the feedback tap to COIL_DRIVE (with appropriate compensation) on the next revision.
- **E/D needs a permanent fix, not just the bench divider.** TI recommends an external current source from V+ sufficient to hold the enable level above the shutdown threshold, plus a 30pF cap from E/D to a low-impedance source for noise immunity.

---

## 5. Consolidated Open Items

**Resolved:**
- [x] ~~5A idle-current symptom~~ — actually caused by the disabled op-amp output stage, not a ground loop (§4.4a)
- [x] ~~Confirm all 5 protection diode polarities~~ — all five confirmed **correct** by physical cathode-band inspection; earlier "D5/D6 backwards" finding retracted (§4.3)
- [x] ~~Verify RV2 pot behavior~~ — working correctly (intentional wiper-to-end tie, full sweep range)
- [x] ~~Diagnose PNP rail pinned at current limit~~ — **OPA455 E/D pin floating at 0.06V → output stage disabled** (§4.4b)
- [x] ~~Fix E/D~~ — 10k/4.7k divider holds E/D at ~4.8V; follower confirmed working (+IN 4V → OUT 4V)

**Open — immediate:**
- [ ] **Bridge SGND to PGND at a single point and re-test** (§4.4c) — the current blocker
- [ ] After the bridge: verify negative rail reaches −15V, PNP ballasts drop to ~0V, bias spread reaches 2.4–2.9V and responds to RV2
- [ ] Scope HV_DRIVE_NODE for oscillation from the C22 capacitive load (§4.5)

**Open — design fixes for next board spin:**
- [ ] Add permanent E/D bias circuit (external current source from V+, or on-board divider) + 30pF noise cap
- [ ] Add a deliberate single-point SGND–PGND tie (net tie / 0Ω / ferrite)
- [ ] Add isolation resistor between OPA455 OUT and C22, or relocate C22
- [ ] Bring Status Flag out to a test point with a 10kΩ pull-up to 5V
- [ ] Reconsider feedback tap location (currently before the output stage — Vbe drops uncorrected)

**Open — deferred:**
- [ ] Confirm thermal coupling of Q12 to the output-device heatsink
- [ ] Verify D1/D2 flyback diode rating against repetitive 44mJ shutdown pulses
- [ ] Isolated (out-of-circuit) test of D1, matching the confirmation already done on D2
- [ ] Bring the design up to full ±30V rails (bring-up currently running at ±15V)
- [ ] Decide whether any Phase 3 low-side MOSFET / IPM H-bridge work is still relevant

---

## 6. Reference Calculations

- **Coil parameters (as measured):** τ = L/R_dc = 1.35ms, R_dc = 0.656Ω → **L ≈ 0.886mH**
- **Stored energy at 10A:** E = ½LI² ≈ **44mJ** per full-current shutdown event
- **Forced discharge time (via D1/D2 + 1000µF rail caps):** dI/dt ≈ V_clamp/L ≈ 34,700 A/s → **≈290µs to zero from 10A**
- **VBE multiplier (Q12) trim math:** M = 1 + R2/(R23+RV2) = 1 + 10000/(2200+RV2); target M ≈ 4–4.8 for a ~2.4–2.9V quiescent spread across the 4-junction loop. **Max pot resistance = minimum bias spread** (safe starting position)
- **E/D divider:** 15V × 4.7k/(10k+4.7k) ≈ **4.8V**; divider current ≈ 1.0mA vs. E/D pin draw ≈ 50µA
- **Ballast resistor current readout:** V across 0.22Ω ÷ 0.22 = device current. 24mV ≈ 109mA; 22mV ≈ 100mA

---

## 7. Debugging Methodology Notes

Lessons worth carrying forward from this bring-up:

1. **Verify polarity physically, not from CAD screenshots.** The retracted D5/D6 finding cost real time. Cathode bands on populated parts, or a direct electrical measurement, are evidence; a triangle glyph read at low resolution is not.
2. **Check that measurements are internally consistent before building a theory on them.** Several dead ends came from numbers that didn't add up (a Vbe of 0.12V alongside a claimed ~29mA of conduction). When the arithmetic disagrees, re-measure before theorizing.
3. **A collapsed rail invalidates everything downstream of it.** Predrive voltages measured while the negative rail sat at −1.4V were consequences, not causes.
4. **Constant voltage across a wide current range = junction clamping; voltage proportional to current = resistive short.** This distinction (1.4V at 0.17A vs 1.2V at 4A) is what ruled out a hard short and redirected the search.
5. **Don't skip "is it even enabled?" on parts with shutdown pins.** The datasheet said a floating E/D self-enables; in practice a 1µA pull-up against a 0.8V threshold does not survive real-world leakage. Read the fine print in the enable/disable section, not just the summary table.
