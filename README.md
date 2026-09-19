# Precision Bias Coil Current Driver

## TL;DR

Bipolar ±10–12.5 A linear current driver for the X/Y/Z/MOT magnetic bias coils of a cold-atom experiment, targeting sub-100 µs response.

- **Legacy prototype recovered:** a MOSFET/IGBT linear current source that had stopped working. Traced the X-axis failure to a dead IGBT gate driver and worked through grounding and thermal issues.
- **Redesigned from scratch:** OPA455 high-voltage op-amp driving a two-stage complementary BJT emitter-follower with 5 paralleled MJL21193G/MJL21194 output pairs.
- **Reverse V_EBO protection:** 5-diode clamp network so the 5 V base-emitter junctions can't be driven into breakdown when the rails collapse unevenly at power-down.
- **Fault 1 (fixed):** the OPA455 enable pin was left floating and measured 0.06 V above E/D Com, inside the disable window, so the output stage was high-impedance. A 10k/4.7k divider now holds E/D at ~4.8 V, and the op-amp tracks correctly (+IN 4 V → OUT 4 V).
- **Fault 2 (root cause identified):** feedback is taken at the op-amp output rather than at COIL_DRIVE. `U2.2 (−IN)`, `U2.6 (OUT)` and `Q13.1 (B)` are the same net, so COIL_DRIVE sits two V_BE (~1.2 V) below COMMAND with nothing to correct it. With the output tied to PGND through the current sensor, the PNP bank sinks current trying to reach −1.2 V, which matches the one-sided conduction and negative-rail collapse seen on the bench.
- **Status:** bring-up at ±15 V; netlist audit complete (§4.6); bench confirmation of the offset model is next.

---

![Layout](https://github.com/DavodChuweiLiao/10-BJT-VCCS-Diagnosis/blob/main/Layout.png?raw=true)
![Schematic](https://github.com/DavodChuweiLiao/10-BJT-VCCS-Diagnosis/blob/main/Screenshot%202026-09-14%20211057.png?raw=true)
![Board](https://github.com/DavodChuweiLiao/10-BJT-VCCS-Diagnosis/blob/main/Screenshot%202026-09-16%20231345.png?raw=true)

**Owner:** Chuwei (David) Liao

**Purpose:** Drive stable, fast-switching, low-ripple current through the X, Y, Z and MOT bias coils of a cold-atom / magneto-optical trap experiment. The command is an analog setpoint voltage. Each axis needs bidirectional current up to ~10–12.5 A with sub-100 µs response.

This document covers the original prototype, the recovery and debug work, and the new linear amplifier design.

---

## 1. Timeline

| Phase | Date | Description |
|---|---|---|
| Original thesis | — | Mithilesh Kumar Parit's PhD thesis sets the target: 500 µs response at 1 A / 2 A steps, 4-axis Helmholtz coil drive |
| Prototype build | documented July 7, 2026 | 4-unit enclosure (X/Y/Z/MOT), MOSFET + IGBT linear VCCS, hand-wired |
| Recovery / debug | 2026 | Unit recovered from storage; Y-axis overheating event, dead X-axis IGBT driver, response times re-measured, low-side redesign explored |
| New amplifier design | current | OPA455 op-amp + discrete complementary BJT push-pull output; PCB fabricated, 2 boards built, in bring-up |

---

## 2. Original Prototype ("Current Stabilization Circuit")

### 2.1 Architecture
High-power linear voltage-controlled current source (VCCS). A series MOSFET operates in its linear region as a voltage-controlled resistor. An external analog PID (SRS LTS-series) closes the loop around a current transducer, holding current constant against coil impedance and thermal drift. Fully linear, no PWM switching noise.

**Signal path:** `+30V → high-side MOSFET (drain) → series IGBT (static on/off switch) → coil → LEM current transducer → PGND`. The transducer secondary (1:250, 4 mA per 1 A primary) drives a 100 Ω burden, giving 0.4 V/A feedback to the PID. The PID output drives the MOSFET gate directly, unbuffered.

### 2.2 Hardware
- 4 units in one metal enclosure: X-Bias, Y-Bias, Z-Bias, MOT
- Through-hole, hand-wired construction
- BNC for signals (Feedback, Measure, PWM, DISABLE); banana plugs for power
- 2 cooling fans; IGBT gate drivers mounted on the enclosure walls

### 2.3 Power / Logic Inputs (per axis)
- **Main power:** +5 V to +30 V drive rail
- **12 V logic** and **5 V logic** for internal circuitry. Swapping 5 V/12 V or reversing polarity destroys the IGBT driver.
- **±15 V:** powers the current transducers. Can be built from two isolated single-channel supplies tied at one common point.
- **As built:** Z-Bias and MOT share a single ±15 V transducer line. Do not connect a second ±15 V supply to these two axes.

### 2.4 Grounding (as built)
- On X-Bias and Z-Bias only, main power return is tied to 12 V logic ground.
- On all axes, the external Measure ground is tied to the transducer's internal ground.
- External supplies must therefore have floating outputs; an earth-referenced supply creates a ground loop through these internal ties.

### 2.5 Absolute Maximum Ratings
| Parameter | Limit |
|---|---|
| Load current | 12.5 A (transducer limit) |
| Main power voltage | 100 V |
| Logic rails | 5 V / 12 V / ±15 V only; over-voltage destroys front-end ICs and IGBT drivers |

**Transfer function:** 0.4 V setpoint = 1 A output (2.5 A/V).

### 2.6 Operating Procedure
1. **Diagnostics:** T-adapter scope taps on Measure and Setpoint.
2. **Power-up:** 12 V, 5 V, ±15 V first, then the PID controller.
3. **PID setup:** clamp PID output to 0–10 V before enabling main power. Known-good settings: P = 0.8, I = 2.6×10⁵, D = 0, Offset = 4.5.
4. **Run:** fans on → main power on → apply setpoint.
5. **Shutdown:** PID off → main power off → logic power off. Never hot-plug the coil, power, or logic.

### 2.7 Protection (as designed)
- Flyback diode across the coil + transducer
- MOV in parallel with the flyback diode for transients beyond the diode rating
- Clamp diode on the transducer output to block negative excursions

### 2.8 Measured Performance
- Rise time 26 µs, fall time 30 µs.
- Consistent with gate-charge limiting: Q_g ≈ 610 nC ÷ 20 mA PID output ≈ 30.5 µs. The unbuffered gate drive is the bottleneck, not the coil or the loop.
- Well beyond the 500 µs thesis target.

### 2.9 Modification
- X-Bias IGBT gate driver (EVAL-ADUM4221-1EBZ) failed, leaving the IGBT stuck off.
- Fix: jumper from MOSFET source to IGBT emitter, removing the IGBT from the current path. X-Bias works normally without the static switch.

### 2.10 BOM
| Component | Part Number | Role |
|---|---|---|
| Linear MOSFET | IXTK60N50L2 | 500 V / 60 A, main control element |
| IGBT | AUIRGPS4070D0 | 400 V / 70 A, static series switch (bypassed on X) |
| IGBT driver | EVAL-ADUM4221-1EBZ | Isolated half-bridge gate driver |
| Current transducer | LEM ITN 12-P ULTRASLAB | 12.5 A, 1:250, closed-loop |
| Burden resistor | 100 Ω precision | 0.4 V/A |
| Reference resistors | 3.3 kΩ | Reference dividers |
| TVS diode | 5KP58A | 5000 W transient suppressor |
| Varistor | V150LA20AP | 150 V RMS MOV |

---

## 3. Recovery & Debug Log

- **Y-axis overheating:** driving a 3.3 kΩ stand-in load with a 770 ms, 0–272 mV square wave (~20 mA) produced smoke after 20 s. Root cause not determined. After re-checking connections, Y was confirmed working (16 ms response at 1 V, P = 0.7).
- **Working axes (Y, Z, MOT):** operating normally at 1 kHz, 405 mV / 0 V, 50% duty.
- **X-axis:** IGBT driver draws 0.47 A and behaves as a short.
- **Response time:** 400 mV → 1 A, 800 mV → 2 A (2.5 A/V confirmed); ~30 µs rise on all axes, matching the gate-charge limit, including with a coil connected.
- **5.5 A steps:** at 3 kHz and 5 kHz, visible phase lag and ringing as the MOSFET gate RC limit is approached; ~10% overshoot, consistent with the thesis.

### 3.1 Redesign Options Considered (superseded by §4)
- **Gate buffer** (BUF634A or LT1210) between PID and MOSFET gate to remove the 30 µs limit; <2.4 µs expected with a 200 mA-class buffer.
- A buffer needs a **low-side MOSFET**; the high-side source floats and can't be driven by a ground-referenced buffer.
- Flyback diode D1 in the original schematic does not match a standard flyback arrangement; unresolved.
- **The IGBT may be unnecessary:** it is never switched in normal operation (PWM/DISABLE jumpered), and removing it saves ~0.6 W/A of conduction loss.
- **H-bridge options evaluated:** Pololu G2 24v21; Infineon IM06B20AC1 IPM (5 samples ordered).
- **LT1210 reference values** (if revisited): non-inverting unity gain, R_F = 750 Ω, C_COMP = 10 nF OUT→COMP, 10 kΩ gate pull-down, 100 nF + 4.7 µF on each rail, 10 Ω gate resistor.

---

## 4. New Linear Amplifier Design (current work)

### 4.1 Topology
Voltage-controlled voltage source: an OPA455 high-voltage op-amp provides gain and accuracy, followed by a two-stage complementary emitter-follower (EF2) BJT output stage for current.

**Signal path:** `COMMAND → OPA455 (+IN) → HV_DRIVE_NODE → Q12 V_BE multiplier → Q13/Q14 drivers → 5 paralleled complementary output pairs → COIL_DRIVE → coil → DP50IP transducer → PGND`

### 4.2 Node Structure (from netlist)

From `BJT_PUSH_PULL.net` (KiCad 10.0.4, 80 components, 46 nets):

```
/HV_DRIVE_NODE  = U2.6 (OUT), U2.2 (-IN), Q13.1 (B), Q12.2 (C),
                  R2.1, C22.2, D5.K, D6.K
```

There is no separate NPN predrive net. The op-amp output, its inverting input, Q13's base and Q12's collector are one node.

V_BE multiplier divider:
```
HV_DRIVE_NODE --R2(10k)-- [X] --R23(2.2k)-- [Y] --RV2(5k trim)-- PNP_PREDRIVE
                            |
                          R26(10Ω)  (Q12 base stopper)
                            |
                        Q12.3 (B)
```
R26 is Q12's base stopper; there is no resistor isolating the op-amp output from the predrive network (see flaw 5).

```
/PNP_PREDRIVE   = Q12.1 (E), Q14.1 (B), C22.1, D5.A, D7.A, R24.1, RV2.2, RV2.3
/NPN_DRIVE_BUS  = Q13.3 (E), R1.1, D3.K, D6.A, R6/R9/R13/R17/R21 (NPN base resistors)
/PNP_DRIVE_BUS  = Q14.3 (E), R1.2, D4.A, D7.K, R3/R10/R14/R18/R22 (PNP base resistors)
/COIL_DRIVE     = J8, D1.A, D2.K, D3.A, D4.K, all ten 0.22 Ω ballasts
```

### 4.3 Stages
- **U2 (OPA455IDDAR):** ±6 V to ±75 V supply, 45 mA output, unity-gain stable. PowerPAD tied to V−, per datasheet.
- **Q12 (BD139) V_BE multiplier:** sets Class-AB bias. R2 = 10k top; R23 (2.2k) + RV2 (5k, 25-turn Bourns 3296W) bottom.
- **Q13/Q14 (MJE15032/MJE15033, TO-220):** complementary driver pair.
- **Q2–Q11 (5× MJL21194 NPN / 5× MJL21193G PNP, TO-264):** output stage; 2.2 Ω base resistors (2512), 0.22 Ω emitter ballasts (TO-220-2).
- **U1 (Danisense DP50IP):** fluxgate transducer, 4 primary turns = 1:250 = 12.5 A nominal. R25 = 100 Ω burden to SGND, output on J13.
- **D1/D2 (TO-220-2, MUR1560G-class, 600 V 15 A):** clamp COIL_DRIVE to the ±30 V rails, backed by 2× 1000 µF per rail.
- **R1 (100 Ω):** bleed between the drive buses (~12 mA).
- **Decoupling:** 2× 1000 µF + 5× 100 nF per main rail; 4.7 µF + 100 nF per op-amp rail.

### 4.4 Reverse V_EBO Protection

MJL21193/94, MJE15032/33 and BD139 all have V_EBO = 5 V despite 250–400 V V_CEO/V_CBO ratings. At power-down, the 4.7 µF op-amp rail caps discharge much faster than the 1000 µF main caps. That can reverse-bias base-emitter junctions beyond 5 V and gradually degrade h_FE over many cycles. Each diode clamps one junction group to a single forward drop.

Orientation, verified from the netlist and the physical cathode bands:

| Diode | Anode | Cathode | Correct |
|---|---|---|---|
| D3 (SS34) | COIL_DRIVE | NPN_DRIVE_BUS | yes |
| D4 (SS34) | PNP_DRIVE_BUS | COIL_DRIVE | yes |
| D5 (1N4148W) | PNP_PREDRIVE | HV_DRIVE_NODE | yes |
| D6 (1N4148W) | NPN_DRIVE_BUS | HV_DRIVE_NODE | yes |
| D7 (1N4148W) | PNP_PREDRIVE | PNP_DRIVE_BUS | yes |

### 4.5 Bring-Up Issue Log

#### 4.5a ~5 A idle current (resolved)
**Symptom:** with all six supply/ground leads connected (no coil, no signal), supply current settled at ~5 A. Removing any one lead dropped it to near zero.

**Cause:** the OPA455 output stage was disabled (§4.5b). With one shared bipolar supply, SGND and PGND were tied through the supply common, so the disabled op-amp's collapsed output pulled the predrive network toward the negative rail and turned the PNP output bank on. Removing any lead broke that path, which looked like a ground loop but was not.

#### 4.5b OPA455 output disabled by floating E/D pin (fixed)
**Symptom:** the negative supply always hit its current limit regardless of RV2 or command. With +30 V off, −30 V still limited; with only +30 V on, nothing happened. Reproduced identically on a second, independently assembled board.

**Ruled out, in order:** Q12 (no shorts); D1/D2 (D2 tested healthy out of circuit); PNP output transistors (B-C-E pinout confirmed against board nets; odd in-circuit readings were parallel paths); RV2 (the pin 2–3 short is an intentional wiper-to-end tie; full sweep normal); D5/D6 orientation (correct, §4.4).

**Finding:** `U2.8 (E/D)` is unconnected, with `U2.1 (E/D Com)` on SGND. Per TI datasheet §7.3.4, a floating E/D self-enables through an internal 1 µA source that holds it ~2 V above E/D Com, only 0.8 V from the disable threshold. TI notes this is not robust.

**Measured:** E/D at 0.06 V above E/D Com, inside the disable range (E/D Com to +0.35 V).

**Supporting evidence:** with +IN at 4 V, both −IN and OUT read 2.5 V: attenuated but tracking. A disabled OPA455 output goes to ~160 kΩ while the input stage stays active, so the output divides against the external network (R2 + R23/RV2 + R24 ≈ 13.5 kΩ).

**Fix (bench):** +15V_OP → 10 kΩ → E/D (pin 8) → 4.7 kΩ → SGND, giving ~4.8 V. Within the 2.5–5 V enable range, below the 7 V abs max, and ~1 mA of divider current against ~50 µA pin draw.

**Result:** with only the signal-side rails powered, +IN = 4 V, −IN = 4 V, OUT = 4 V.

#### 4.5c Negative rail still collapses, PNP bank conducting at zero command (root cause identified, bench confirmation pending)

| Measurement | Value | Interpretation |
|---|---|---|
| −15 V rail at 0.17 A limit | −1.4 V | supply in CC mode |
| Rail vs current (0.17 A → 4 A) | 1.4 V → 1.2 V | nearly constant over 20× current: junction clamp, not a resistive short |
| Supply terminal vs board pad | no drop | cabling ruled out |
| PNP_DRIVE_BUS → −15 V (unpowered) | 0.5 MΩ | no short |
| COIL_DRIVE → +15 V / −15 V (unpowered) | 800 kΩ / 80 kΩ | asymmetric, too high to matter |
| COIL_DRIVE → PGND (unpowered) | 460 kΩ | no board short; the tie is the external test loop |
| V across R1 (100 Ω) | 0.389 V → 3.9 mA | no driver shoot-through |
| PNP ballasts (0.22 Ω) | 3 of 5 at 24 mV, later all 5 at 13 mV | PNP output bank carries the load |
| NPN ballasts | 0–0.032 mV (≤145 µA) | NPN bank off |
| HV_DRIVE_NODE → COIL_DRIVE | +0.960 V | see open question below |
| PNP_PREDRIVE → COIL_DRIVE | −0.875 V | |
| NPN_DRIVE_BUS → COIL_DRIVE | +0.60 V | NPN output at the knee |
| PNP_DRIVE_BUS → COIL_DRIVE | −0.58 V | PNP output conducting |
| RV2 full sweep, PNP_DRIVE_BUS | −0.58 → −0.59 V | see note below |
| D2 removed | no change | D2 not involved |
| SGND → PGND | 200 kΩ | grounds not bonded (flaw 3) |

**Root cause: feedback point.** Since `U2.2 (−IN)`, `U2.6 (OUT)` and `Q13.1 (B)` are one net, the op-amp regulates HV_DRIVE_NODE rather than the output. COIL_DRIVE sits two V_BE below it, uncorrected:

```
COIL_DRIVE ≈ COMMAND − V_BE(Q13) − V_BE(Q_npn) ≈ COMMAND − 1.2 V
```

At COMMAND = 0 V the amplifier's operating point is COIL_DRIVE = −1.2 V. On the bench, COIL_DRIVE is tied to PGND through J8 → coil/short → J9 → DP50IP primary → PGND. Held ~1.2 V above where it wants to be, the stage sinks current through the PNP bank toward the negative rail.

This is consistent with every observation: PNP on and NPN off; current flowing PGND → COIL_DRIVE → PNP → −15 V; only the negative rail collapsing; rail voltage clamped at a junction drop; RV2 unable to fix it (it sets bias spread, not offset); identical behavior on both boards (a design issue, not damage).

With the real 0.656 Ω coil, a −1.2 V offset means ~1.8 A of uncommanded current at zero setpoint. This is a design defect, not a tuning issue.

**Note on RV2:** a 10 mV shift at PNP_DRIVE_BUS over a full RV2 sweep looks negligible, but V_BE is logarithmic in current: 10 mV ≈ e^(10/26) ≈ 1.5× change in output-stage current. RV2 is working. Bias should be trimmed by watching ballast voltage (linear in current), not junction voltage.

**Open question:** HV_DRIVE_NODE read +0.96 V relative to COIL_DRIVE with COMMAND at 0 V. A working follower with SGND and PGND bonded should give 0 V. Most likely this was measured before the SGND–PGND bridge was added. Re-measure HV_DRIVE_NODE → PGND directly at COMMAND = 0 V; if it reads ~+1 V, there is an additional problem.

**Verification test (no hardware change):** sweep COMMAND up from 0 V. Near +1.2 V the natural operating point of COIL_DRIVE reaches 0 V, so the current should drop toward zero and the negative rail should recover.

**Fix:** move `U2.2 (−IN)` from HV_DRIVE_NODE to COIL_DRIVE so the output stage is inside the loop. This removes the DC offset and also corrects crossover distortion, V_BE thermal drift and output impedance. Requires feedback compensation, since the output stage adds phase shift inside the loop.

---

### 4.6 Netlist Design Audit

Review of `BJT_PUSH_PULL.net`. Severity: **Critical**, **High**, **Medium**, **Note**.

#### 1. [Critical] Feedback taken at op-amp output, not at the load
`U2.2 (−IN)` = `U2.6 (OUT)` = `Q13.1 (B)` = `/HV_DRIVE_NODE`; the output stage is outside the loop.
**Effects:** ~1.2 V uncorrected offset (≈1.8 A at zero command into 0.656 Ω); uncorrected crossover distortion; V_BE thermal drift as the heatsink warms; output impedance not reduced by loop gain.
**Fix:** take −IN from COIL_DRIVE and add compensation. This is the cause of the current bring-up failure.

#### 2. [Critical] DP50IP transducer supply shared with ±30 V op-amp rails
`U1.10` (+15 V pin) is on `/+30V_OP` and `U1.12` (−15 V pin) is on `/-30V_OP`, the same nets as the OPA455 supplies. The DP50IP supply range is ±14.25 V to ±15.75 V. Raising the rails above that for more output swing will destroy the transducer. It has survived only because bring-up is at ±15 V.
**Fix:** give U1 its own regulated ±15 V, separate from the op-amp rails.

#### 3. [Critical] NT1 net tie has no footprint; SGND and PGND are not bonded
The schematic includes the single-point ground tie (`NT1.1` on `/SGND`, `NT1.2` on `/PGND`), but the footprint field is empty:
```
(comp (ref "NT1") (value "NetTie_2") ... (field (name "Footprint") )
```
Nothing was placed on the PCB, so the grounds are unconnected in copper, matching the measured 200 kΩ.
**Fix:** assign a `NetTie_2` footprint or fit a 0 Ω link. Until the respin, keep one external single-point bridge.

#### 4. [High] E/D pin floating (bodged, §4.5b)
`unconnected-(U2-E_D-Pad8)`. **Permanent fix:** per TI, an external current source from V+ holding E/D above the shutdown threshold plus 30 pF from E/D to a low-impedance node; a stiff divider as bodged also works.

#### 5. [High] C22 (1 µF) directly on the op-amp output
`C22.2` is on `/HV_DRIVE_NODE` = `U2.6 (OUT)`. The OPA455 is rated for 200 pF capacitive load; this is 5000× that with no series resistance.
**Effect:** reduced phase margin and likely HF oscillation, previously hidden because the output was disabled.
**Fix:** 10–47 Ω in series between OUT and C22, or relocate C22. Scope HV_DRIVE_NODE once the amp is driving; a DMM's AC range will not see oscillation in the 100 kHz–MHz range.

#### 6. [High] V_BE multiplier divider current too low
Divider current ≈ (spread − V_BE)/R2 ≈ 1.75 V / 10k ≈ 0.18 mA. Q12 collector current from R24 is ~13 mA at ±15 V and ~27 mA at ±30 V; with h_FE ≈ 100–250, base current is 0.05–0.28 mA. The ratio is under 2× (should be ≥10×) and can invert at ±30 V.
**Effect:** bias spread depends on Q12's h_FE, varying between units and with temperature.
**Fix:** scale the divider down ~10× (R2 = 1k, R23 = 220 Ω, RV2 = 500 Ω) for ~1.8 mA, keeping the same trim range.

Trim range is adequate as is:
```
M = 1 + R2/(R23 + RV2) = 1 + 10k/(2.2k + 0…5k)  ⇒  M = 2.39 … 5.55
spread = M × V_BE(Q12) ≈ 1.55 V … 3.6 V     (target ≈ 2.4–2.6 V for 4 junctions)
target at RV2 ≈ 1.1–1.5k (~25% of travel)
```

#### 7. [High] Q12 not thermally coupled to the output devices
For the V_BE multiplier to track output-stage temperature, Q12 should be mounted on the output heatsink.

#### 8. [High] COMMAND has no DC return
`/COMMAND` = `J1.1`, `C21.1` (100 pF to SGND), `U2.3 (+IN)`, with no resistor to ground. With J1 unplugged, +IN floats and the output is undefined.
**Fix:** 10k–100k from COMMAND to SGND.

#### 9. [High] Status Flag unconnected
`unconnected-(U2-Status_Flag_N-Pad5)`. Over-temperature and over-current put the output in high impedance, which looks the same as E/D shutdown.
**Fix:** 10 kΩ pull-up to 5 V (referenced to E/D Com) and a test point.

#### 10. [Medium] D1/D2 not specified
Value field is `"D"`, description `"DIODE GEN PURP 600V 15A TO220-2 (e.g., MUR1560G)"`, with no committed part.
Check repetitive surge rating against 44 mJ per shutdown, and recovery time. D1/D2 are also the only thing defining COIL_DRIVE when unloaded.

#### 11. [Medium] No Zobel network at the output
`/COIL_DRIVE` has only the ballasts, four diodes and J8. Class-AB stages driving inductive loads typically need an output R-C for HF stability.
**Fix:** 10 Ω + 100 nF from COIL_DRIVE to PGND.

#### 12. [Medium] No DC bleed on COIL_DRIVE
COIL_DRIVE floats with no coil connected. A 10k bleed to PGND defines it for unloaded bring-up.

#### 13. [Medium] No output current limiting
Output devices rely on the bench supply's current limit. For a 10 A design, consider V-I limiter transistors across the drivers.

#### 14. [Medium] Burden resistor at the transducer maximum
R25 = 100 Ω; the DP50IP allows 0–100 Ω. No margin for tolerance or temperature. At 1:250, 10 A → 40 mA → 4.0 V; 12.5 A → 5.0 V.

#### 15. [Note] NPN parts use the PNP footprint
`MJL21194` parts are assigned `MJL21193G:TO545P2030X530X2900-3`. Same TO-264 package and B-C-E pinout, so likely fine; verify the pin map once.

#### 16. [Note] RV2 is 25-turn
Bourns 3296W needs 25 turns end to end. The measured 3 kΩ maximum on a 5 kΩ part suggests end of travel was not reached; check endpoints with an ohmmeter.

#### 17. [Note] Power dissipation
- 0.22 Ω ballasts at 2 A each (10 A ÷ 5): 0.88 W each; TO-220-2, needs heatsinking.
- R24 (1k axial) at ±30 V: 0.76 W in a ~1 W part; use 2 W.
- 2.2 Ω base resistors: ~3.5 mW, fine.
- R1: ~14 mW, fine.

#### Verified correct
- All five protection diodes correctly oriented (§4.4)
- Base-resistor / ballast pairings correct on all ten devices
- 5 NPN + 5 PNP balanced; collectors on the correct rails
- Symmetric decoupling: 2× 1000 µF + 5× 100 nF per main rail
- OPA455 PowerPAD on V−
- RV2 wiper tied to one end (fail-safe against wiper open)
- DP50IP wired 4 turns = 1:250 = 12.5 A nominal

---

## 5. Open Items

**Resolved**
- [x] ~5 A idle current: caused by the disabled op-amp (§4.5a)
- [x] Protection diode polarity confirmed (§4.4)
- [x] RV2 functional (§4.5c)
- [x] OPA455 E/D floating: found and bodged (§4.5b)
- [x] Netlist audit (§4.6)

**Next bench session**
- [ ] Measure HV_DRIVE_NODE → PGND at COMMAND = 0 V (expect ~0 V)
- [ ] Sweep COMMAND to ~+1.2 V; confirm current drops (verifies offset model)
- [ ] Confirm the external SGND–PGND bridge is single-point
- [ ] Scope HV_DRIVE_NODE for oscillation (C22 load)
- [ ] Trim RV2 using ballast voltages

**Next board revision**
- [ ] Critical: move −IN feedback to COIL_DRIVE, add compensation
- [ ] Critical: separate DP50IP ±15 V supply from op-amp rails
- [ ] Critical: footprint for NT1 (or 0 Ω link)
- [ ] High: permanent E/D bias + 30 pF
- [ ] High: series resistor between OPA455 OUT and C22
- [ ] High: scale V_BE multiplier divider ~10× (R2 1k / R23 220 Ω / RV2 500 Ω)
- [ ] High: move Q12 to the output heatsink
- [ ] High: DC return on COMMAND
- [ ] High: Status Flag pull-up + test point
- [ ] Medium: real part number for D1/D2, verify surge rating
- [ ] Medium: Zobel at COIL_DRIVE
- [ ] Medium: DC bleed on COIL_DRIVE
- [ ] Medium: output current limiting
- [ ] Medium: re-evaluate R25

**Deferred**
- [ ] Raise rails to ±30 V, only after the DP50IP supply is separated
- [ ] Test D1/D2 under repetitive 44 mJ shutdown pulses
- [ ] Out-of-circuit test of D1
- [ ] Decide whether any §3.1 MOSFET / H-bridge work is still relevant

---

## 6. Reference Calculations

- **Coil:** τ = L/R = 1.35 ms, R = 0.656 Ω → L ≈ 0.886 mH
- **Stored energy at 10 A:** ½LI² ≈ 44 mJ per shutdown
- **Discharge through D1/D2 into 1000 µF:** dI/dt ≈ 34,700 A/s → ~290 µs from 10 A to zero
- **V_BE multiplier:** M = 2.39 … 5.55 → spread 1.55 … 3.6 V; target 2.4–2.6 V at RV2 ≈ 1.1–1.5k. Maximum pot resistance gives minimum spread (safe starting point).
- **Divider vs base current:** 0.18 mA vs 0.05–0.28 mA → ratio < 2× (want ≥ 10×)
- **Current output offset:** COIL_DRIVE ≈ COMMAND − 1.2 V → ≈ 1.8 A at 0 V command into 0.656 Ω
- **E/D divider:** 15 V × 4.7k / 14.7k ≈ 4.8 V; ~1.0 mA vs ~50 µA pin draw
- **Ballast readout:** I = V / 0.22 Ω; 13 mV ≈ 59 mA, 24 mV ≈ 109 mA
- **Transducer scaling:** 1:250 with 100 Ω → 0.4 V/A; 10 A → 4.0 V, 12.5 A → 5.0 V
- **V_BE sensitivity:** 10 mV ≈ 1.5× current change at room temperature

---

## 7. Debug Notes

1. **Constant voltage across a wide current range indicates junction clamping; voltage proportional to current indicates a resistive short.** 1.4 V at 0.17 A vs 1.2 V at 4 A ruled out a hard short.
2. **Trim bias on a linear observable.** Ballast voltage is proportional to current; V_BE is logarithmic and hides a 1.5× change in 10 mV.
3. **A collapsed rail invalidates downstream measurements.** Predrive voltages taken with the negative rail at −1.4 V reflect the collapse, not its cause.
4. **Check enable/shutdown pins first on parts that have them.** A 1 µA internal pull-up against a 0.8 V margin is not a reliable enable.
5. **The test setup is part of the circuit.** COIL_DRIVE was tied to PGND through the sense path for most of this debug, which is the worst-case load for a Class-AB stage and changed the symptoms.
6. **Verify polarity from the netlist, the physical part, or a measurement.** Schematic symbols and footprint graphics are not sufficient on their own.
