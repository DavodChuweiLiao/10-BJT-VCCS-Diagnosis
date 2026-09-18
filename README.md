# Precision Bias Coil Current Driver — Project README

## 🔑 TL;DR

**Precision ±10–12.5A analog current driver for magnetic bias coils in a cold-atom/MOT physics experiment, sub-100µs response.**

- 🔧 **Diagnosed & recovered** a dead legacy MOSFET/IGBT current-source prototype — traced a failed IGBT gate driver, resolved ground-loop and thermal issues
- 🔁 **Redesigned from scratch**: precision op-amp (OPA455) + discrete BJT push-pull output stage, replacing the old MOSFET/IGBT topology
- 🛡️ **Added a 5-diode protection network** to prevent transistor degradation from reverse-bias stress during power cycling
- ✅ **Fault #1 SOLVED — floating OPA455 E/D pin.** Measured **0.06V** above E/D Com, inside the disable window; output stage sat at 160kΩ. Fixed with a 10k/4.7k divider holding E/D at ~4.8V; op-amp now tracks correctly (+IN 4V → OUT 4V).
- 🔴 **Fault #2 ROOT CAUSE — feedback is taken at the op-amp output, not at COIL_DRIVE.** `U2.2 (−IN)`, `U2.6 (OUT)` and `Q13.1 (B)` are all the *same net*, so COIL_DRIVE idles **2 Vbe (~1.2V) below COMMAND** with no way for the loop to correct it. With the output shorted to PGND through the current sensor, the PNP bank sinks current trying to reach −1.2V — exactly the observed one-sided conduction and rail collapse.
- 🚧 **Status:** running at ±15V for bring-up; full netlist audit complete (§4.6), bench verification of the offset theory pending

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
| New amplifier redesign | current | OPA455-based op-amp + discrete complementary BJT push-pull (EF2) output stage; PCB fabricated, 2 boards built, under bring-up |

---

## 2. Phase 1 & 2 — Original Prototype ("Current Stabilization Circuit")

### 2.1 Architecture
High-power linear **Voltage-Controlled Current Source (VCCS)**. A series MOSFET operates in its linear region as a voltage-controlled variable resistor; an external analog PID (SRS LTS-series controller) closes the loop around a current transducer measurement to hold current constant regardless of coil impedance or thermal drift. No PWM switching noise — pure linear control.

**Signal path:** `+30V supply → high-side MOSFET (drain) → series IGBT (static ON/OFF switch) → coil load → LEM current transducer → PGND`. Transducer secondary (1:250 ratio, 4mA per 1A primary) drives a 100Ω burden resistor, producing 0.4V/A feedback to the external PID. PID output drives the MOSFET gate directly (unbuffered).

### 2.2 Hardware Configuration
- 4 physical units in one metal enclosure: **X-Bias, Y-Bias, Z-Bias, MOT**
- Through-hole breadboard construction, hand-wired
- BNC connectors for signals (Feedback, Measure, PWM, DISABLE); banana plugs for power
- 2 cooling fans; IGBT gate drivers mounted on enclosure walls

### 2.3 Power/Logic Inputs (per axis)
- **Main Power (Red/Black):** +5V to +30V drive rail
- **12V Logic:** for internal circuitry
- **5V Logic:** for internal circuitry — ⚠️ swapping 5V/12V or reversing polarity destroys the IGBT driver
- **±15V Bipolar:** powers the analog current transducers (can be synthesized from two isolated single-channel supplies, tied at a common GND point)
- **As-built quirk:** Z-Bias and MOT share a single ±15V line for their transducers — do not wire a second, conflicting ±15V supply to these two axes.

### 2.4 As-Built Grounding Architecture (critical — caused real issues)
- On **X-Bias and Z-Bias only**: Main Power negative return (Power−) is physically tied to 12V logic ground.
- On **all axes**: the external "Measure" ground is internally tied to the current transducer's internal ground.
- Because of the above, external power supplies **must have floating outputs** — otherwise an earth-ground path plus the internal ties creates a ground loop.

### 2.5 Absolute Maximum Ratings
| Parameter | Limit |
|---|---|
| Max load current | **12.5A** (hard limit of the internal current transducer) |
| Max main power voltage | **100V** |
| Logic rails | Strictly 5V / 12V / ±15V — over-voltage destroys front-end ICs and IGBT drivers |

**Transfer function:** 1A output = 0.4V setpoint input (gain = 2.5 A/V)

### 2.6 Standard Operating Procedure
1. **Diagnostics:** T-adapter oscilloscope taps on Measure and Setpoint lines.
2. **Power-up:** energize 12V, 5V, ±15V first; then the external PID controller.
3. **PID config:** clamp output to 0–10V *before* energizing main power. Known-good baseline: **P = 0.8, I = 2.6×10⁵, D = 0, Offset = 4.5.**
4. **Active operation:** fans on → energize Main Power → apply setpoint.
5. **Shutdown (in order):** PID controller off → Main Power off → all logic/peripheral power off. Never hot-plug the coil, power, or logic while energized.

### 2.7 Protection Circuitry (as designed)
- **Flyback diode** across the coil + current-transducer series combination
- **Varistor (MOV)** in parallel with the flyback diode, clamping transients beyond the diode's reverse breakdown
- **Ground-clamping diode** tying the transducer output to ground, preventing negative excursions that would damage the transducer

### 2.8 Measured Performance (Phase 2 report, July 2026)
- Rise time: 26µs, Fall time: 30µs
- Matches theory: MOSFET gate charge Qg ≈ 610nC ÷ PID's 20mA max output = **30.5µs theoretical minimum** — confirms the unbuffered gate drive is the bottleneck, not the coil or control loop.
- Thesis target was 500µs @ 1–2A — prototype greatly exceeded this.

### 2.9 As-Built Modification
- **X-Bias axis IGBT gate driver (EVAL-ADUM4221-1EBZ) failed** — IGBT stuck open (OFF).
- **Fix applied:** bypass jumper soldered from MOSFET Source directly to IGBT Emitter, removing the IGBT from the current path. X-Bias operates normally without its static switch.

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

- **Y-axis heating incident:** driving a 3.3kΩ "coil" stand-in with a 770ms/0–272mV square wave (~20mA) produced smoke after 20s. Root cause not resolved in the log; connections re-checked, Y subsequently confirmed working (16ms response @ 1V, P=0.7).
- **Baseline good-axis behavior (Y, Z, MOT):** working at setpoint 1kHz, 405mV high / 0V low, 50% duty.
- **X-axis fault confirmed:** IGBT driver draws 0.47A and behaves like a short circuit.
- **Response time re-verification:** 400mV → 1A, 800mV → 2A (matches 2.5A/V); ~30µs rise time across axes, matching the unbuffered-gate limit — confirmed even with a coil connected.
- **5.5A step-response replication:** square wave at 3kHz and 5kHz — visible phase lag and oscillation approaching the RC limit of the MOSFET gate. ~10% overshoot on transitions, consistent with the thesis.

### 3.1 Redesign Directions Explored (superseded by Phase 4, kept for reference)
- **Gate buffer (BUF634A or LT1210)** between PID output and MOSFET gate to eliminate the 30µs charge-time bottleneck (theoretical <2.4µs with a 200mA-class buffer).
- **Buffer requires a low-side MOSFET topology** — a high-side MOSFET has a floating source, which a ground-referenced buffer can't drive.
- **Open question at the time:** the flyback diode D1 in the original schematic didn't match a standard flyback topology; flagged to the thesis author, no reply in the log.
- **IGBT necessity questioned:** no experimental procedure ever toggled the IGBT as a switch; PWM/DISABLE pins appeared hardwired via jumpers. Removing it would save ~0.6W/A of conduction loss.
- **H-bridge candidates evaluated:** Pololu G2 24v21; Infineon IM06B20AC1 IPM (5 samples ordered).
- **LT1210 buffer reference design** (if this path is revisited): non-inverting gain-of-1, RF = 750Ω (1%, 1/4W metal film), CCOMP = 10nF OUT→COMP, RPD = 10kΩ gate pull-down, CBYP 100nF + CBULK 4.7µF on both rails, 10Ω gate resistor.

**Status:** Superseded by the Phase 4 linear amplifier redesign.

---

## 4. Phase 4 — New Linear VCVS Amplifier Redesign (current work)

### 4.1 Topology
A **voltage-controlled voltage source (VCVS)**: precision high-voltage op-amp (OPA455) for gain/accuracy, buffered by a **two-stage complementary emitter-follower (EF2)** discrete BJT output stage for current capacity.

**Signal path:** `COMMAND → OPA455 (+IN) → HV_DRIVE_NODE → Q12 VBE-multiplier bias network → Q13/Q14 driver pair → 5× paralleled complementary output pairs → COIL_DRIVE → coil → DP50IP current transducer → PGND`

### 4.2 Actual Node Structure (from netlist — differs from earlier assumptions)

Verified against `BJT_PUSH_PULL.net` (KiCad Eeschema 10.0.4, 80 components, 46 nets):

```
/HV_DRIVE_NODE  = U2.6 (OUT), U2.2 (-IN), Q13.1 (B), Q12.2 (C),
                  R2.1, C22.2, D5.K, D6.K
```

**There is no separate NPN_PREDRIVE net.** The op-amp output, the op-amp's own inverting input, Q13's base, and Q12's collector are all one node. This single fact drives most of §4.6.

The VBE multiplier divider chain:
```
HV_DRIVE_NODE --R2(10k)-- [X] --R23(2.2k)-- [Y] --RV2(5k trim)-- PNP_PREDRIVE
                            |
                          R26(10Ω)  ← base stopper only
                            |
                        Q12.3 (B)
```
**R26 is a base stopper for Q12, not an isolation resistor between the op-amp and the predrive node** (an earlier assumption that turned out wrong, and it matters — see flaw #5).

```
/PNP_PREDRIVE   = Q12.1 (E), Q14.1 (B), C22.1, D5.A, D7.A, R24.1, RV2.2, RV2.3
/NPN_DRIVE_BUS  = Q13.3 (E), R1.1, D3.K, D6.A, R6/R9/R13/R17/R21 (NPN base resistors)
/PNP_DRIVE_BUS  = Q14.3 (E), R1.2, D4.A, D7.K, R3/R10/R14/R18/R22 (PNP base resistors)
/COIL_DRIVE     = J8, D1.A, D2.K, D3.A, D4.K, all ten 0.22Ω ballasts
```

### 4.3 Key Stages
- **U2 (OPA455IDDAR):** high-voltage precision op-amp, ±6V to ±75V rated, 45mA output, unity-gain stable. PowerPAD tied to V− ✓ correct per datasheet.
- **Q12 (BD139) VBE multiplier:** sets Class-AB quiescent bias. R2 = 10k top, R23 (2.2k) + RV2 (5k, 25-turn Bourns 3296W) bottom.
- **Q13/Q14 (MJE15032/MJE15033, TO-220):** first-stage complementary driver pair.
- **Q2–Q11 (5× MJL21194 NPN / 5× MJL21193G PNP, TO-264):** paralleled output stage, 2.2Ω base resistors (2512 SMD), 0.22Ω emitter ballasts (TO-220-2 power resistors).
- **U1 (Danisense DP50IP):** fluxgate current transducer, wired for **4 primary turns = 1:250 ratio = 12.5A nominal**. R25 = 100Ω burden to SGND, output to J13 BNC.
- **D1/D2 (TO-220-2, spec'd as MUR1560G-class 600V 15A):** COIL_DRIVE clamped to ±30V rails, backed by 2× 1000µF per rail.
- **R1 (100Ω):** driver-stage bleed between the two drive buses (~12mA).
- **Decoupling:** 2× 1000µF + 5× 100nF on each main rail (symmetric ✓); 4.7µF + 100nF on each op-amp rail.

### 4.4 Reverse VEBO Protection — all five diodes VERIFIED CORRECT

MJL21193/94, MJE15032/33, and BD139 all have **VEBO = 5V** despite 250–400V VCEO/VCBO. During power cycling the small op-amp caps (4.7µF) collapse far faster than the 1000µF main caps, which can reverse-bias base-emitter junctions past 5V and cumulatively degrade hFE.

Orientation confirmed **directly from the netlist** (definitive — supersedes all earlier screenshot-based analysis):

| Diode | Anode net | Cathode net | Required anode | ✓ |
|---|---|---|---|---|
| D3 (SS34) | COIL_DRIVE | NPN_DRIVE_BUS | COIL_DRIVE | ✅ |
| D4 (SS34) | PNP_DRIVE_BUS | COIL_DRIVE | PNP_DRIVE_BUS | ✅ |
| D5 (1N4148W) | PNP_PREDRIVE | HV_DRIVE_NODE | PNP_PREDRIVE | ✅ |
| D6 (1N4148W) | NPN_DRIVE_BUS | HV_DRIVE_NODE | NPN_DRIVE_BUS | ✅ |
| D7 (1N4148W) | PNP_PREDRIVE | PNP_DRIVE_BUS | PNP_PREDRIVE | ✅ |

> ⚠️ **RETRACTED FINDING — methodology note.** An earlier analysis based on reading diode triangle glyphs in low-resolution KiCad footprint screenshots concluded D5 and D6 were backwards and blamed them for the bring-up fault. **This was wrong**, and it cost hours. Physical cathode-band inspection contradicted it, and the netlist now settles it definitively. **Lesson: a pixel-level read of a CAD symbol is not evidence.** Use the netlist, the physical cathode band, or a direct measurement.

### 4.5 Board Bring-Up — Issue Log

#### 4.5a Initial 5A idle current (RESOLVED — misdiagnosed at the time)
**Symptom:** with all six power/ground connections made (no coil, no signal), PSU current settled at ~5A; disconnecting any single lead restored near-zero current.

**Original hypothesis (wrong):** a ground loop from sharing one bipolar PSU across SGND and PGND.

**Actual cause:** the OPA455's output stage was disabled (§4.5b). With one shared bipolar PSU, SGND and PGND *were* tied through the supply common, so the disabled op-amp's collapsed output dragged the predrive network to the negative rail and turned the PNP output bank hard on. Unplugging any lead broke that path, mimicking ground-loop behaviour.

#### 4.5b Op-amp output stage disabled — floating E/D pin (✅ ROOT CAUSE CONFIRMED AND FIXED)
**Symptom:** negative rail always pulls PSU current limit regardless of RV2 or command. With +30V off, −30V still maxes. With only +30V on, nothing happens. **Reproduced identically on a second, independently assembled board** with verified-good components.

**Candidates ruled out in order:** Q12 (no shorts) → D1/D2 (D2 isolated, tested healthy) → PNP output transistors (pinout confirmed B-C-E against board nets; suspicious in-circuit readings were parallel-path artifacts) → RV2 (pin 2–3 "short" is an intentional wiper-to-end fail-safe tie; full sweep confirmed) → D5/D6 orientation (retracted, §4.4) → **E/D pin**.

**The finding:** the schematic leaves `U2.8 (E_D)` **unconnected**, with `U2.1 (E_D_Com)` tied to SGND. TI §7.3.4 says a floating E/D self-enables — but only via an internal **1µA** source holding it ~2V above E/D Com, against a disable threshold only 0.8V away. TI explicitly warns this is fragile.

**Measured: E/D sat at 0.06V above E/D Com** — squarely inside the disable window (spec: disabled = E/D Com to +0.35V).

**Confirming signature:** with +IN at 4V, both −IN and OUT read **2.5V** — attenuated but tracking. That is the textbook disabled-output behaviour: output impedance rises to ~160kΩ while the inputs stay active, so the signal divides against the external network (R2 + R23/RV2 + R24 ≈ 13.5kΩ). A dead amp gives a rail; an enabled follower gives a clean 4V.

**Fix applied (bench bodge):** +15V_OP → **10kΩ** → E/D (pin 8) → **4.7kΩ** → SGND ⇒ **~4.8V** on E/D. Mid-window (enable = 2.5–5V), under the 7V E/D-to-E/D-Com abs max, ~1mA of pull-up vs the pin's ~50µA draw.

**Result: ✅ CONFIRMED.** With only signal-side rails powered: **+IN = 4V, −IN = 4V, OUT = 4V.** The follower closes its loop.

#### 4.5c Negative rail still collapses — PNP bank conducting at zero command (🔴 ROOT CAUSE IDENTIFIED, bench verification pending)

**Measurement summary:**

| Measurement | Value | Interpretation |
|---|---|---|
| −15V rail under 0.17A limit | −1.4V | supply in CC mode, output collapsed |
| Rail vs current (0.17A → 4A) | 1.4V → 1.2V | **near-constant across 20× current** → junction clamping, not a resistive short |
| Supply terminal vs board pad | no drop | rules out cable/connector resistance |
| PNP_DRIVE_BUS → −15V (unpowered) | 0.5 MΩ | no short |
| COIL_DRIVE → +15V / −15V (unpowered) | 800kΩ / 80kΩ | asymmetric but far too high to matter |
| **COIL_DRIVE → PGND (unpowered)** | **460 kΩ** | no board short — the tie is the external test loop |
| V across R1 (100Ω) | 0.389V → 3.9mA | **no driver shoot-through** |
| PNP ballasts (0.22Ω) | 3 of 5 at 24mV, later all 5 at 13mV | **PNP output bank is the load** |
| NPN ballasts | 0 to 0.032mV (145µA) | NPN bank essentially off |
| HV_DRIVE_NODE → COIL_DRIVE | +0.960V | ⚠️ see inconsistency below |
| PNP_PREDRIVE → COIL_DRIVE | −0.875V | |
| NPN_DRIVE_BUS → COIL_DRIVE | +0.60V | output NPN at the knee |
| PNP_DRIVE_BUS → COIL_DRIVE | −0.58V | output PNP conducting |
| RV2 full sweep effect on PNP_DRIVE_BUS | −0.58 → −0.59V | see reinterpretation below |
| D2 cut out of circuit | no change | **D2 eliminated** |
| SGND → PGND resistance | **200 kΩ** | grounds not bonded (flaw #3) |

**ROOT CAUSE: the feedback point.** Because `U2.2 (−IN)` and `U2.6 (OUT)` are the same net as `Q13.1 (B)`, the op-amp servos **HV_DRIVE_NODE**, not the output. COIL_DRIVE therefore sits two Vbe below it with no correction:

```
COIL_DRIVE ≈ COMMAND − Vbe(Q13) − Vbe(Q_npn) ≈ COMMAND − 1.2 V
```

With COMMAND = 0V, the amplifier's natural operating point is **COIL_DRIVE = −1.2V**. But COIL_DRIVE is shorted to PGND (0V) through J8 → coil/short → J9 → the DP50IP primary → PGND. The output is being held ~1.2V *above* where it wants to be, so the amplifier **sinks** current to pull it down — and sinking is precisely the PNP bank pulling toward the negative rail.

**This explains every observation:** PNP conducting / NPN off ✓ · current flowing PGND → COIL_DRIVE → PNP → −15V ✓ · negative rail collapsing while positive doesn't ✓ · rail voltage clamped at a junction drop rather than scaling with current ✓ · RV2 unable to fix it (it sets the *spread*, not the *offset*) ✓ · reproducing identically on both boards ✓ (design-level, not damage).

**With the real 0.656Ω coil this is not a bench artifact** — a −1.2V offset across 0.656Ω is **≈1.8A of uncommanded current at zero setpoint.** For a precision current driver that is a fundamental design defect, not a tuning problem.

**Reinterpretation of the "RV2 has no authority" result:** the 10mV shift measured at PNP_DRIVE_BUS across a full pot sweep looked like nothing, but a junction voltage is **logarithmic** in current — 10mV at room temperature is a factor of e^(10/26) ≈ **1.5× change in output-stage current**. RV2 *was* working; the measurement was simply the wrong observable. **Measure the ballast resistors (linear in current) when trimming, never the junction voltages.**

**Unresolved inconsistency to check first thing next session:** HV_DRIVE_NODE measured **+0.96V** relative to COIL_DRIVE while COMMAND was driven to 0V. A working follower with SGND and PGND bonded should put HV_DRIVE_NODE at 0V. Likely explanation: that measurement was taken before the SGND–PGND bridge, so the two references were floating apart. **Re-measure HV_DRIVE_NODE → PGND directly with COMMAND at 0V.** It must read ~0V; if it reads ~+1V, there is a further problem beyond the offset.

**Verification test (no hardware changes):** sweep COMMAND upward from 0V. Around **+1.2V**, COIL_DRIVE's natural operating point reaches 0V and matches the short, so **the current should collapse toward zero and the negative rail should come up.** This confirms or refutes the offset model in five minutes.

**Fix:** move the feedback tap — `U2.2 (−IN)` from HV_DRIVE_NODE to **COIL_DRIVE**. The op-amp then servos HV_DRIVE_NODE up by whatever the Vbe drops require, and COIL_DRIVE tracks COMMAND directly. This simultaneously fixes the DC offset, crossover distortion, Vbe thermal drift, and output impedance. Requires a feedback compensation cap, since the output stage is now inside the loop.

---

### 4.6 🔬 Full Netlist Design Audit

Systematic review of `BJT_PUSH_PULL.net`. Severity: 🔴 critical · 🟠 high · 🟡 medium · ⚪ note.

#### 🔴 1. Feedback taken at op-amp output, not at the load
`U2.2 (−IN)` = `U2.6 (OUT)` = `Q13.1 (B)` = `/HV_DRIVE_NODE`. Output stage is entirely outside the feedback loop.
**Effects:** ~1.2V (2 Vbe) uncorrected DC offset → **≈1.8A error at zero command into 0.656Ω**; crossover distortion uncorrected; Vbe thermal drift uncorrected (output wanders as the heatsink warms); output impedance not reduced by loop gain.
**Fix:** tap −IN at COIL_DRIVE, add feedback compensation. *This is the dominant defect and the cause of the current bring-up failure.*

#### 🔴 2. DP50IP current transducer will be destroyed at ±30V
`U1.10` is named **+15V** in the symbol but is wired to `/+30V_OP`; `U1.12` (**−15V**) is wired to `/-30V_OP`. Those nets are **shared with the OPA455 supply pins**.
Danisense DP50IP absolute supply range: <cite index="19-1">±14.25V ~ ±15.75V</cite>. The OPA455 is happy to ±75V, so the design clearly intends to raise these rails — and the moment they go above ~±15.75V, a ~$600 transducer dies.
**Why it hasn't happened yet:** bring-up is currently at ±15V.
**Fix:** split the rails. Give U1 its own regulated ±15V; run the op-amp section on whatever the output swing requires. Never share the net.

#### 🔴 3. NT1 net tie has NO FOOTPRINT — SGND and PGND are not bonded
The schematic *does* contain the intended single-point ground bridge (`NT1.1` on `/SGND`, `NT1.2` on `/PGND`), but the netlist shows:
```
(comp (ref "NT1") (value "NetTie_2") ... (field (name "Footprint") )  ← EMPTY
```
No footprint ⇒ nothing placed on the PCB ⇒ **the two grounds were never connected in copper.** This is exactly the measured 200kΩ, and it is why the op-amp section had no defined reference relative to the power section.
**Fix:** assign a `NetTie_2` footprint (or a 0Ω link). Until respun, keep the external single-point bridge — **one point only**.

#### 🟠 4. E/D pin left floating *(already found and bodged — §4.5b)*
`unconnected-(U2-E_D-Pad8)`. **Permanent fix:** TI recommends an external current source from V+ sufficient to hold the enable level above the shutdown threshold, plus a 30pF cap from E/D to a low-impedance node for noise immunity. A stiff divider (as bodged) also works.

#### 🟠 5. C22 (1µF) hangs directly on the op-amp output with zero isolation
`C22.2` is on `/HV_DRIVE_NODE` = `U2.6 (OUT)`. The OPA455 is rated `CLOAD Capacitive load drive` = **200pF**. This is **5000× over spec**, and because R26 turned out to be Q12's base stopper rather than an output isolation resistor, **there is nothing in series at all**.
**Effects:** phase-margin collapse, likely HF oscillation — masked until now because the output stage was disabled.
**Fix:** series resistor (10–47Ω) between OUT and C22, or move C22 so it doesn't load the op-amp output directly. **Scope HV_DRIVE_NODE as soon as the amp is driving.** A handheld DMM in AC mode cannot detect this (oscillation will be 100kHz–MHz, past most meters' AC bandwidth).

#### 🟠 6. VBE multiplier divider current is too low relative to Q12's base current
Divider current ≈ (spread − Vbe)/R2 ≈ 1.75V / 10k ≈ **0.18mA**.
Q12's collector current is set by R24: at ±15V ≈ 13mA, at ±30V ≈ 27mA. With BD139 hFE ≈ 100–250, base current ≈ **0.05–0.28mA**.
The standard design rule is divider current ≥ **10×** base current. Here the ratio is **below 2×**, and at ±30V it can invert.
**Effect:** the bias spread becomes strongly dependent on Q12's hFE (unit-to-unit spread and temperature), so the trim drifts and is hard to set reproducibly.
**Fix:** scale the divider down ~10× — e.g. R2 = 1k, R23 = 220Ω, RV2 = 500Ω — giving ~1.8mA of divider current. Keep the same ratios so the trim range is preserved.

**Trim range itself is adequate** (an earlier suggestion to change R23 was wrong and is withdrawn):
```
M = 1 + R2/(R23 + RV2) = 1 + 10k/(2.2k + 0…5k)  ⇒  M = 2.39 … 5.55
spread = M × Vbe(Q12) ≈ 1.55V … 3.6V     (target ≈ 2.4–2.6V for 4 junctions)
target lands at RV2 ≈ 1.1–1.5k, i.e. ~25% of travel
```

#### 🟠 7. Q12 is not thermally coupled to the output devices
Q12 is a TO-126 vertical in the mid-left signal cluster; the output devices are TO-264 on the top-edge heatsink. A VBE multiplier only compensates thermal drift if it **tracks the temperature of the devices it biases**.
**Effect:** as the output stage heats at 10A, its Vbe falls, quiescent current rises, which heats it further — classic thermal runaway. The 0.22Ω ballasts help but are not a substitute.
**Fix:** relocate Q12 onto the output heatsink (or bond it thermally) in the next spin.

#### 🟠 8. COMMAND has no DC return
`/COMMAND` = `J1.1`, `C21.1` (100pF to SGND), `U2.3 (+IN)`. **No resistor to ground.** If J1 is ever unplugged, +IN floats on 30pA of bias current and the output can wander anywhere.
**Fix:** 10k–100k from COMMAND to SGND.

#### 🟠 9. Status Flag unconnected
`unconnected-(U2-Status_Flag_N-Pad5)`. Overtemperature and overcurrent both produce a high-impedance output — the *same* signature as E/D shutdown, which is exactly the ambiguity that slowed this debug.
**Fix:** 10kΩ pull-up to 5V (referenced to E/D Com) plus a test point.

#### 🟡 10. D1/D2 are generic and under-specified
Value field is just `"D"`, description `"DIODE GEN PURP 600V 15A TO220-2 (e.g., MUR1560G)"` — no committed part number.
**Needed:** verify repetitive surge rating against the 44mJ-per-shutdown flyback energy, and confirm recovery speed. Note D1/D2 are also the *only* things defining COIL_DRIVE's potential when the output is unloaded.

#### 🟡 11. No Zobel network or output snubber
Class-AB stages driving inductive loads commonly need an R-C series network at the output for HF stability. `/COIL_DRIVE` contains only the ten ballasts, four diodes, and connector J8 — nothing else.
**Fix:** add a Zobel (e.g. 10Ω + 100nF to PGND) at COIL_DRIVE.

#### 🟡 12. No DC bleed on COIL_DRIVE
With no coil connected, COIL_DRIVE floats. A high-value bleed (10k to PGND) would define it for safe unloaded bring-up.

#### 🟡 13. No output-stage current limiting
Output devices are protected only by the external PSU's current limit. For a 10A design, consider VI-limiter transistors across the drivers.

#### 🟡 14. Burden resistor at the transducer's maximum allowed value
R25 = 100Ω; DP50IP spec is <cite index="25-1">Measuring resistance RM Ω 0 100</cite>. Operating exactly at the maximum leaves no margin for tolerance or temperature.
At the wired 1:250 ratio, 10A → 40mA secondary → 4.0V across R25 (12.5A → 5.0V).

#### ⚪ 15. NPN devices use the PNP part's footprint library
`MJL21194` (NPN) parts are assigned `MJL21193G:TO545P2030X530X2900-3`. Same TO-264 package and both are B-C-E, so this is very likely harmless — but worth a one-time pin-map verification since it's a library cross-reference.

#### ⚪ 16. RV2 is a 25-turn trimmer
Bourns 3296W. "Fully clockwise" requires **25 revolutions** — the measured 3kΩ maximum on a 5kΩ part suggests the end of travel was never actually reached during bring-up. Verify the true endpoints with an ohmmeter rather than by feel.

#### ⚪ 17. Power-dissipation notes
- 0.22Ω ballasts at 2A each (10A ÷ 5) = **0.88W each** — TO-220-2 package, needs heatsinking.
- R24 (1k axial) at ±30V = **0.76W** in a ~1W part — marginal; consider 2W.
- 2.2Ω base resistors (2512) ≈ 3.5mW — fine.
- R1 (100Ω) ≈ 14mW — fine.

#### ✅ What the audit found correct
- All five protection diodes correctly oriented (§4.4)
- All ten base-resistor / ballast-resistor pairings correct and consistent
- 5 NPN + 5 PNP balanced; collectors correctly on their respective rails
- Decoupling symmetric and generous: 2× 1000µF + 5× 100nF per main rail
- OPA455 PowerPAD correctly tied to V−
- RV2 wiper tied to one end (standard fail-safe against wiper wear)
- DP50IP wired for 4 turns = 1:250 = 12.5A nominal, appropriate for the 10A target

---

## 5. Consolidated Open Items

**Resolved:**
- [x] 5A idle current — actually the disabled op-amp, not a ground loop (§4.5a)
- [x] All five diode polarities — confirmed correct from netlist; earlier "backwards" finding retracted (§4.4)
- [x] RV2 behaviour — working; the 10mV observation was a log-scale measurement artifact (§4.5c)
- [x] **OPA455 E/D floating → output disabled** — found and bodged (§4.5b)
- [x] Full netlist design audit (§4.6)

**Next bench session, in order:**
- [ ] Re-measure **HV_DRIVE_NODE → PGND** with COMMAND at 0V (must be ~0V) — resolves the +0.96V inconsistency
- [ ] **Sweep COMMAND to ~+1.2V** and confirm the current collapses — verifies the offset model
- [ ] Verify the external SGND–PGND bridge is still single-point
- [ ] Scope HV_DRIVE_NODE for oscillation (C22 capacitive load)
- [ ] Trim RV2 by watching **ballast voltages**, not junction voltages

**Design fixes for next spin (priority order):**
- [ ] 🔴 Move feedback tap from HV_DRIVE_NODE to COIL_DRIVE + add compensation
- [ ] 🔴 Separate the DP50IP ±15V supply from the op-amp rails
- [ ] 🔴 Assign a footprint to NT1 (or fit a 0Ω link)
- [ ] 🟠 Permanent E/D bias circuit + 30pF noise cap
- [ ] 🟠 Isolation resistor between OPA455 OUT and C22
- [ ] 🟠 Scale the VBE-multiplier divider down ~10× (R2 1k / R23 220Ω / RV2 500Ω)
- [ ] 🟠 Relocate Q12 to the output heatsink for thermal tracking
- [ ] 🟠 10k–100k DC return on COMMAND
- [ ] 🟠 Status Flag pull-up + test point
- [ ] 🟡 Commit a real part number for D1/D2 and verify surge rating
- [ ] 🟡 Add Zobel network at COIL_DRIVE
- [ ] 🟡 Add COIL_DRIVE DC bleed resistor
- [ ] 🟡 Consider output-stage current limiting
- [ ] 🟡 Re-evaluate R25 = 100Ω (at the transducer's RM maximum)

**Deferred:**
- [ ] Bring rails to full ±30V — **only after the DP50IP supply is separated**
- [ ] Verify D1/D2 against repetitive 44mJ shutdown pulses
- [ ] Isolated out-of-circuit test of D1 (D2 already done)
- [ ] Decide whether any Phase 3 MOSFET / IPM H-bridge work remains relevant

---

## 6. Reference Calculations

- **Coil:** τ = L/R_dc = 1.35ms, R_dc = 0.656Ω → **L ≈ 0.886mH**
- **Stored energy at 10A:** E = ½LI² ≈ **44mJ** per shutdown
- **Forced discharge via D1/D2 + 1000µF:** dI/dt ≈ 34,700 A/s → **≈290µs to zero from 10A**
- **VBE multiplier:** M = 1 + R2/(R23+RV2) = 1 + 10k/(2.2k + 0…5k) = **2.39 … 5.55** ⇒ spread ≈ **1.55 … 3.6V**; target ≈ 2.4–2.6V at RV2 ≈ 1.1–1.5k. **Max pot resistance = minimum spread** (safe start)
- **Multiplier divider current:** ≈ 0.18mA vs Q12 base current 0.05–0.28mA → **ratio < 2× (want ≥ 10×)**
- **Output offset (current design):** COIL_DRIVE ≈ COMMAND − 1.2V ⇒ **≈1.8A error at 0V command into 0.656Ω**
- **E/D divider:** 15V × 4.7k/(10k+4.7k) ≈ **4.8V**; ~1.0mA vs ~50µA pin draw
- **Ballast readout:** V / 0.22 = device current. 13mV ≈ 59mA; 24mV ≈ 109mA
- **Transducer scaling:** 1:250, R25 = 100Ω ⇒ **0.4 V/A**; 10A → 4.0V, 12.5A → 5.0V
- **Junction voltages are logarithmic:** 10mV of Vbe change ≈ **1.5× current change** — always trim on ballast voltage, not Vbe

---

## 7. Debugging Methodology Notes

1. **Verify polarity from the netlist or the physical part, never from a CAD screenshot.** The retracted D5/D6 finding cost hours.
2. **Check internal consistency before building a theory.** Several dead ends came from numbers that didn't add up (a 0.12V Vbe alongside a claimed 29mA of conduction; 295mA through the ballasts while the supply delivered 170mA).
3. **A collapsed rail invalidates everything downstream.** Predrive voltages measured with the negative rail at −1.4V were consequences, not causes.
4. **Constant voltage across a wide current range = junction clamping; voltage proportional to current = resistive short.** 1.4V at 0.17A vs 1.2V at 4A is what ruled out a hard short.
5. **Know which observables are linear and which are logarithmic.** Trimming bias by watching a junction voltage hides a 1.5× current change in 10mV. Ballast resistors are linear; use those.
6. **Don't skip "is it even enabled?" on parts with shutdown pins.** The datasheet said a floating E/D self-enables; a 1µA pull-up against a 0.8V threshold does not survive real-world leakage.
7. **Get the netlist early.** One parse answered questions that days of in-circuit probing could not — including three latent bugs that had nothing to do with the fault being chased.
8. **Test-setup conditions are part of the circuit.** COIL_DRIVE was shorted to PGND through the sense path for much of this debug, which is the worst-case load for a class-AB stage and materially changed the symptoms.
