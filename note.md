# STM32 Solar MPPT Charge Controller

## System Specification

**Document status:** Preliminary design baseline  
**Revision:** 0.1  
**Date:** 2026-09-13

## 1. Purpose and Scope

This document specifies the Version 1 STM32-based solar MPPT charge controller for charging one 12 V, 18 Ah VRLA battery from one 100 W photovoltaic (PV) module.

The controller shall use an asynchronous buck power stage, Perturb and Observe (P&O) MPPT, and three-stage battery charging (constant current, constant voltage, float). Hardware and firmware protection functions shall take priority over MPPT operation.

Unless explicitly marked **Preliminary**, values in this document are design requirements. Preliminary values require schematic review, prototype measurement, and component-datasheet verification before release.

## 2. External Equipment

### 2.1 PV Module

| Parameter                               |                Specification |
| --------------------------------------- | ---------------------------: |
| Model                                   | Walton 100 W mono-PERC panel |
| Rated maximum power                     |                        100 W |
| Voltage at maximum power, \(V_{MP}\)    |                      21.05 V |
| Current at maximum power, \(I_{MP}\)    |                       4.75 A |
| Open-circuit voltage at STC, \(V_{OC}\) |                      24.48 V |
| Short-circuit current, \(I_{SC}\)       |                       5.17 A |

### 2.2 Battery

| Parameter                  |                                    Specification |
| -------------------------- | -----------------------------------------------: |
| Model                      |                                   Walton WBU1218 |
| Chemistry                  |                          VRLA / sealed lead-acid |
| Nominal voltage            |                                             12 V |
| Capacity                   |                            18 Ah at 20-hour rate |
| Maximum charging current   |                                            5.4 A |
| Cycle-use voltage at 25 °C |                                      14.4–15.0 V |
| Float/standby voltage      | Manufacturer data to be confirmed before release |

## 3. Functional Requirements

### 3.1 Power Conversion

| Requirement                    |                                 Specification |
| ------------------------------ | --------------------------------------------: |
| Topology                       |                   Asynchronous buck converter |
| Input source                   |        One PV module specified in Section 2.1 |
| PV operating/design envelope   |                                    18–25 V DC |
| Power-stage output envelope    |                                10.5–14.4 V DC |
| Maximum charge current         |                       5.0 A, firmware-limited |
| Maximum regulated charge power |                      72 W at 14.4 V and 5.0 A |
| Power-stage capability         |                                   100 W class |
| Switching frequency            |                          100 kHz, preliminary |
| Target converter efficiency    |        ≥90% over the intended charging region |
| Nominal operating point        | 21.05 V PV input, 14.4 V output, 5.0 A output |

The PV and output ranges above are power-stage design envelopes, not absolute source or battery limits. In particular, 10.5 V represents a deeply discharged battery condition, not a normal charging target. Normal battery charging regulation is defined by the CC/CV/Float algorithm and battery requirements.

The controller is not required to transfer the full 100 W panel rating to the battery. The 5.0 A charge-current limit caps battery charging power at 72 W at 14.4 V.

### 3.2 Control and Charging

1. The controller shall implement P&O MPPT using measured PV voltage and current.
2. The controller shall implement CC → CV → Float charging.
3. The CC current limit shall be 5.0 A.
4. The initial CV target shall be 14.4 V at 25 °C.
5. The float-voltage setpoint shall not be finalized until the applicable battery manufacturer requirement is confirmed.
6. Under insufficient PV power, the controller shall use the available PV power without attempting to force the 5.0 A charge-current limit.
7. Control priority shall be:
   1. protection and fault handling;
   2. battery charging limits;
   3. MPPT control; and
   4. PWM duty-cycle command.

### 3.3 PWM

| Requirement                             |        Specification |
| --------------------------------------- | -------------------: |
| PWM source                              |          STM32 timer |
| Nominal frequency                       |              100 kHz |
| Nominal duty cycle at 21.05 V to 14.4 V |                68.4% |
| Preliminary duty-cycle design envelope  | approximately 42–80% |
| Sustained 100% duty                     |           Prohibited |

The high-side bootstrap driver requires periodic switch-node transitions to refresh its bootstrap supply. Firmware shall impose a maximum duty cycle below 100% and preserve adequate off-time.

## 4. Electrical Architecture

### 4.1 Power Path

```text
PV+ → Input fuse → Reverse-polarity protection → TVS clamp → CIN
    → Q1 high-side MOSFET → SW → L1 → BAT+

System GND → D1 Schottky freewheel diode → SW

BAT− → battery-current shunt → system return
PV−  → PV-current shunt      → system return
```

Q1 is the only actively switched power device. D1 conducts naturally during Q1 off-time.

### 4.2 Control and Measurement Path

```text
STM32F103C8T6
  ├─ PWM → LM5106 high-side gate driver → Q1
  ├─ ADC ← PV voltage divider
  ├─ ADC ← PV-current amplifier
  ├─ ADC ← Battery voltage divider
  ├─ ADC ← Battery-current amplifier
  └─ ADC ← Optional power-stage NTC
```

The gate driver shall be disabled on a protection fault.

## 5. Measurement Requirements

### 5.1 MCU and ADC

| Parameter         |                                      Specification |
| ----------------- | -------------------------------------------------: |
| MCU               |                                      STM32F103C8T6 |
| ADC resolution    |                                             12 bit |
| ADC reference     |                                              3.3 V |
| Required channels |   \(V_{PV}\), \(I_{PV}\), \(V_{BAT}\), \(I_{BAT}\) |
| Optional channel  |                            Power-stage temperature |
| Sampling          | Periodic, synchronized with switching as practical |

Firmware shall calculate:

\[
P_{PV}=V_{PV}I_{PV}
\]

\[
P_{BAT}=V_{BAT}I_{BAT}
\]

\[
\eta=\frac{P_{BAT}}{P_{PV}}\times100\%
\]

### 5.2 Voltage Measurement Baseline

| Channel         | Divider       | Filter           | Input range | ADC voltage at maximum input |
| --------------- | ------------- | ---------------- | ----------: | ---------------------------: |
| PV voltage      | 91 kΩ / 12 kΩ | 100 nF to ground |      0–25 V |                       2.91 V |
| Battery voltage | 56 kΩ / 15 kΩ | 100 nF to ground |      0–15 V |                       3.17 V |

Divider resistors shall be 1% tolerance or better. Firmware shall support calibration using measured divider ratios and ADC-reference error.

### 5.3 Current Measurement Baseline

| Parameter            |                                  Specification |
| -------------------- | ---------------------------------------------: |
| Current-sense method |    Low-side shunt with current-sense amplifier |
| Number of channels   |            Two: PV current and battery current |
| Shunt resistance     |                        10 mΩ each, preliminary |
| Shunt rating         |                                      ≥1 W each |
| Amplifier            |             INA180A2, gain 50 V/V, preliminary |
| Output scaling       | \(V_{ADC}=0.5I\) V; therefore \(I=2V_{ADC}\) A |
| ADC filter           |      1 kΩ series resistor and 100 nF capacitor |

At 5.0 A battery current, the nominal current-sense amplifier output is 2.5 V. The approximately 5.67 A inductor-current peak is used for Q1, D1, and L1 sizing; it is not directly equal to the battery current measured by the battery shunt. The output capacitor carries the inductor-current AC component.

Both current shunts shall use Kelvin sense connections. The final PCB shall define a controlled star/return topology so shunt voltage drops do not corrupt voltage sensing or gate-driver ground references.

## 6. Protection Requirements

| Condition                   | Required response                       | Initial threshold / status                      |
| --------------------------- | --------------------------------------- | ----------------------------------------------- |
| PV input overvoltage        | Disable PWM and gate driver             | Threshold TBD during protection design          |
| PV/input overcurrent        | Limit current or disable PWM            | Threshold TBD during protection design          |
| Battery overvoltage         | Reduce or disable charging              | Threshold TBD during protection design          |
| Battery overcurrent         | Reduce PWM; disable on persistent fault | >5.0 A                                          |
| Output short circuit        | Fast current limit or shutdown          | Threshold and response time to be verified      |
| Power-stage overtemperature | Derate, then shutdown                   | Derate at 70 °C; shutdown at 85 °C, preliminary |
| Reverse PV connection       | Prevent damage                          | Hardware protection required                    |
| Reverse battery connection  | Prevent damage                          | Hardware protection required                    |
| MCU/software fault          | Disable switching safely                | Watchdog and fault-safe driver disable required |
| Startup                     | Controlled duty ramp                    | Soft-start required                             |

The 14.4 V CV target is a regulation setpoint, not an overvoltage-fault threshold. The battery overvoltage threshold shall be set above the CV target after battery limits, measurement error, ripple, control-loop overshoot, and protection margin are evaluated.

The PV design envelope is not an overvoltage-protection threshold. The final PV overvoltage threshold shall account for cold-condition \(V_{OC}\), expected operating behavior, and available semiconductor/clamp margin.

The input and battery fuses are backup protection; they shall not be used as normal current-control elements.

## 7. Preliminary Power-Stage Design Baseline

### 7.1 Calculated Operating Values

| Item                                                             |                             Value |
| ---------------------------------------------------------------- | --------------------------------: |
| Maximum required input current at 72 W output and 90% efficiency |   approximately 3.82 A at 21.05 V |
| Recommended input current capability                             |                              ≥6 A |
| Inductor-ripple design target                                    |   1.5 A peak-to-peak (30% of 5 A) |
| Worst evaluated ripple point                                     |           25 V input, 12 V output |
| Calculated inductance at worst evaluated point                   |                           41.6 µH |
| Selected preliminary inductance                                  |                             47 µH |
| Ripple at 25 V input, 12 V output with 47 µH                     | approximately 1.33 A peak-to-peak |
| Full-load inductor peak current                                  |              approximately 5.67 A |
| Full-load inductor RMS current                                   |              approximately 5.02 A |

The converter is expected to operate in continuous-conduction mode at full load.

### 7.2 Preliminary Component Requirements

| Reference                      | Baseline specification                                                                  | Status                |
| ------------------------------ | --------------------------------------------------------------------------------------- | --------------------- |
| Q1                             | STP55NF06, N-channel MOSFET, 60 V, TO-220, approximately 15 mΩ at 10 V gate drive       | Preliminary candidate |
| D1                             | STPS10L60, 60 V, 10 A Schottky, TO-220AC                                                | Preliminary candidate |
| L1                             | 47 µH shielded power inductor; ≥5 A RMS, ≥8 A saturation current, DCR preferably ≤30 mΩ | Exact part TBD        |
| Gate driver                    | LM5106 high-side bootstrap driver                                                       | Preliminary candidate |
| Gate resistor                  | 10 Ω                                                                                    | Tune on prototype     |
| Gate-source resistor           | 10 kΩ                                                                                   | Preliminary           |
| Bootstrap capacitor            | 100 nF X7R, 100 V preferred                                                             | Preliminary           |
| Bootstrap diode                | Low-capacitance Schottky, 40–60 V minimum                                               | Exact part TBD        |
| Input bulk capacitor           | 100 µF, 50 V, low ESR                                                                   | Preliminary           |
| Input ceramic capacitors       | 2 × 10 µF, 50 V, X7R                                                                    | Preliminary           |
| Input high-frequency capacitor | 100 nF, 50 V ceramic                                                                    | Preliminary           |
| Output bulk capacitor          | 470 µF, 25 V, low ESR                                                                   | Preliminary           |
| Output ceramic capacitors      | 2 × 22 µF, 25 V, X7R                                                                    | Preliminary           |
| PV current shunt               | 10 mΩ, ≥1 W                                                                             | Preliminary           |
| Battery current shunt          | 10 mΩ, ≥1 W                                                                             | Preliminary           |
| Current-sense amplifiers       | 2 × INA180A2                                                                            | Preliminary           |
| Temperature sensor             | 10 kΩ NTC near Q1/D1                                                                    | Preliminary           |
| PV fuse                        | 6 A or 6.3 A                                                                            | Preliminary           |
| Battery fuse                   | 7.5 A                                                                                   | Preliminary           |

### 7.3 Auxiliary Supplies

| Rail  | Load                     | Requirement                                  |
| ----- | ------------------------ | -------------------------------------------- |
| 10 V  | LM5106 gate driver       | Regulated from PV input; exact regulator TBD |
| 3.3 V | STM32 and analog sensing | Regulated supply; exact regulator TBD        |

A resistor divider shall not be used to power the gate driver. The selected regulators shall tolerate the PV input range and expected transients.

## 8. PCB and Layout Requirements

1. The `CIN–Q1–D1` high-\(di/dt\) loop shall be as small as practical.
2. Input ceramic capacitors shall be placed adjacent to the Q1/D1 power loop.
3. The SW-node copper area shall be minimized to reduce ringing and EMI.
4. Output capacitors shall be placed close to L1 and the output return path.
5. Current-shunt sense traces shall be Kelvin-routed and kept away from the SW node.
6. Analog sensing ground, gate-driver ground, and high-current returns shall join at a defined low-impedance reference point.
7. Q1 and D1 shall have sufficient copper area for heat spreading. The D1 footprint shall support an optional heatsink.

## 9. Preliminary Performance and Thermal Targets

| Operating point          | Estimated converter loss | Estimated efficiency | Status                   |
| ------------------------ | -----------------------: | -------------------: | ------------------------ |
| 21.05 V to 14.4 V at 5 A |                2.8–3.4 W | approximately 94–96% | Analytical estimate only |
| 25 V to 12 V at 5 A      |                3.5–4.5 W |    approximately 93% | Analytical estimate only |

At the nominal operating point, estimated losses are approximately 0.8 W in Q1, 0.77 W in D1, 0.5–0.76 W in L1 copper loss for 20–30 mΩ DCR, and 0.39 W in the two shunts combined.

These estimates exclude or simplify several layout-dependent effects, including switch-node ringing, actual semiconductor switching transitions, auxiliary-rail losses, capacitor ESR, and thermal-interface performance. Prototype measurements are required before claiming compliance with the efficiency or temperature targets.

## 10. Verification Requirements

The design shall not be released for procurement or field use until the following are verified. Semiconductor voltage stress is a critical prototype measurement because 60 V ratings do not represent 60 V of usable operating margin; switching-node ringing and TVS clamp behavior must be included.

| Verification item            | Acceptance criterion                                                                               |
| ---------------------------- | -------------------------------------------------------------------------------------------------- |
| PV input voltage margin      | Confirm cold-condition \(V_{OC}\), cable effects, and transient clamp margin                       |
| Semiconductor voltage stress | Measured Q1 and D1 voltage remains below derated device limits with ringing included               |
| Inductor                     | Verified saturation current, DCR, core loss, and temperature rise at maximum load                  |
| Gate drive                   | Correct gate voltage, bootstrap refresh, rise/fall times, and no false turn-on                     |
| Current sensing              | Calibrated readings and no ADC clipping over the required current range                            |
| Voltage sensing              | Calibrated readings and no ADC clipping at maximum voltage                                         |
| Battery charging             | CC, CV, float behavior, and temperature compensation policy verified against battery documentation |
| Fault handling               | Each protection condition disables or limits switching safely                                      |
| Thermal                      | Q1, D1, L1, shunts, and capacitors remain within rated temperature limits                          |
| Efficiency                   | ≥90% across representative PV and battery operating points                                         |
| EMI and layout               | Switching-node ringing and conducted/radiated noise assessed on the prototype                      |

## 11. Open Items

1. Confirm battery float-voltage specification and charging temperature-compensation requirements.
2. Determine maximum PV open-circuit voltage over the intended ambient-temperature range. The 25 V input limit is based on STC panel data and is not yet a verified absolute maximum.
3. Select the final input TVS device and verify that its clamp voltage protects 60 V Q1 and D1 ratings under transient conditions.
4. Select and validate the reverse-polarity MOSFET circuit.
5. Select exact 10 V and 3.3 V regulators.
6. Select exact inductor, capacitors, bootstrap diode, and fuse part numbers after availability and thermal review.
7. Verify INA180A2 bandwidth, input filtering, offset, gain accuracy, and transient behavior for the final sensing implementation.
8. Define the final hardware fault thresholds, filtering, hysteresis, and recovery behavior.
9. Measure switching waveforms and tune the gate resistor as needed.

## 12. Recommended Verification Equipment

| Measurement                   | Recommended equipment                                                   |
| ----------------------------- | ----------------------------------------------------------------------- |
| PV and battery voltage        | Calibrated digital multimeter                                           |
| PV and battery current        | Digital multimeter, current probe, or shunt measurement                 |
| Gate waveform and PWM         | Oscilloscope with appropriate probe                                     |
| Switching-node waveform       | Oscilloscope with appropriately rated probe and short ground connection |
| Ripple and transient response | Oscilloscope                                                            |
| Temperature                   | Thermocouples, temperature logger, or IR camera                         |
| Efficiency                    | Calibrated voltage/current meters or power analyzer                     |
| MPPT performance              | Controlled illumination or repeatable variable-PV test conditions       |

## 13. Change Control

Any change to the PV module, battery type, maximum charging current, switching frequency, power topology, or semiconductor voltage rating requires review of the control limits, component stress, protection thresholds, thermal design, and verification plan.
