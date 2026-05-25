# EV CAN Bus Signal Integrity: EMI Mitigation under High-Voltage Transients

[![Simulation](https://img.shields.io/badge/Simulation-LTspice-blue.svg)]()
[![Protocol](https://img.shields.io/badge/Protocol-CAN_Bus-green.svg)]()
[![Event](https://img.shields.io/badge/Event-UDYAM_'26-orange.svg)]()
[![Author](https://img.shields.io/badge/Author-Arka_Kar-purple.svg)]()

This repository contains the LTspice simulation files, physical models, and comparative data analysis for mitigating severe electromagnetic interference (EMI) on a Controller Area Network (CAN) bus operating within an Electric Vehicle (EV) environment. 

This project was developed for the **CONTINUUM PS-2 | UDYAM '26** engineering problem statement.

## 📌 Executive Summary
Modern Electric Vehicles rely on the CAN bus (ISO 11898) to communicate critical data between the Electronic Control Units (ECUs), Battery Management System (BMS), and motor controllers. However, this fragile 1 Mbps data network is frequently routed adjacent to high-voltage traction inverters. 

Modern EV inverters utilize Silicon Carbide (SiC) MOSFETs to switch hundreds of volts in nanoseconds (e.g., 100V pulses with 10ns edge rates). This violent switching generates aggressive $dV/dt$ and $dI/dt$ transients. Through parasitic capacitive coupling, these transients inject severe common-mode voltage spikes onto the CAN bus, destroying differential signal integrity and causing catastrophic Bit Error Rate (BER) failure.

**Project Objective:** Implement, simulate, and compare hardware mitigation strategies to suppress peak common-mode voltage deviation to strictly **< 100mV** while maintaining a BER of **$\le 10^{-9}$**.

---

## 🔬 The Physics of the Problem

The failure of the unmitigated CAN bus is driven by two fundamental physical phenomena modeled in this simulation:

1. **Capacitive Coupling:** The physical proximity of the high-voltage motor cables to the unshielded CAN wires creates a parasitic air-gap capacitor. Because capacitive impedance decreases as frequency increases ($Z_c = \frac{1}{2\pi f C}$), the ultra-fast 10ns switching edges of the motor act as high-frequency AC, easily crossing the gap.
2. **Inductive Kickback:** Once the transient current enters the CAN data lines, it encounters the natural parasitic inductance of the copper wiring harness (modeled as $0.5\mu\text{H}$ per segment). The inductors resist this sudden surge in current, generating massive, destructive voltage spikes governed by Faraday's law of induction:
   
   $$V_{spike} = L \frac{di}{dt}$$

---

## 📂 Repository Structure
* `/simulations/` - Contains the raw LTspice `.asc` circuit files.
  * `baseline_model.asc` - Unmitigated Unshielded Twisted Pair (UTP) CAN bus.
  * `shielding_model.asc` - Approach A: Physical Shielding via Faraday Cage.
  * `rc_filter_model.asc` - Approach B: Passive RC Low-Pass Filter network.
* `/images/` - Contains oscilloscope waveform exports and schematic captures.
* `UDYAM_Report.pdf` - The comprehensive mathematical and technical documentation.

---

## ⚠️ Baseline System Architecture (Unmitigated)

The baseline model simulates a standard Unshielded Twisted Pair (UTP) harness under attack by a traction inverter.

### System Parameters
* **Transceiver Logic:** Differential signaling between 1.5V (Dominant) and 3.5V (Recessive), with a 2.5V common-mode resting state.
* **Harness Parasitics:** Modeled with series inductance of **0.5µH** and series resistance of **5Ω** per segment. The inter-wire differential capacitance is **100pF**, representing ~2.5 meters of standard twisted copper.
* **The Aggressor:** A 100V switching transient with 10ns $t_r / t_f$.
* **Termination:** Standard $120\Omega$ resistors at both extremities to prevent transmission line reflections.

**Result:** The baseline fails catastrophically. Severe inductive ringing destroys the logic thresholds, confirming a complete failure of the $\le 10^{-9}$ BER requirement.

![Baseline Waveform](images/baseline_image.png) 
> *Figure 1: Unmitigated baseline showing severe common-mode deviation and destroyed differential signaling.*

---

## 🛡️ Mitigation Strategies

### Approach A: Shielded Twisted Pair (STP)
This approach simulates the introduction of a grounded, braided copper shield wrapping the CAN wires.
* **Implementation:** Two **47nF** shunt capacitors were added from CAN_H and CAN_L to the chassis ground, immediately preceding the transceiver termination.
* **Mechanism of Action:** The grounded copper braid physically intercepts the external electric field (Faraday Cage effect). Because high-frequency noise seeks the path of least impedance, the motor transients are shunted harmlessly to ground through the shield's natural capacitance to ground before they can develop a voltage across the $120\Omega$ termination.
* **Performance:** Peak common-mode deviation clamped to **~75mV**. Edge rates remain crisp and near-ideal.

### Approach B: RC Low-Pass Filter
A lightweight, cost-effective PCB-level alternative utilized when strict EV weight budgets prohibit heavy copper shielding.
* **Implementation:** A **10Ω** series resistor on both CAN lines, followed by a **470pF** shunt capacitor to ground.
* **Mechanism of Action:** This network forms a passive low-pass filter with a cutoff frequency of:
  
  $$f_c = \frac{1}{2 \pi R C} \approx 33.8 \text{ MHz}$$
  
  The 10ns motor transients consist of high-frequency harmonics well above 30 MHz, causing the capacitors to act as short-circuits to ground. The slower 1 Mbps CAN data (fundamental frequency ~500 kHz) falls well below the cutoff, passing through to the transceiver with minimal attenuation.
* **Performance:** Peak common-mode deviation clamped to **~70mV**. Ringing is heavily damped.

---

## 📊 Comparative Analysis & Metrics

The simulations utilize LTspice `.meas` commands to mathematically extract the peak absolute deviation from the 2.5V baseline:
`.meas TRAN CommonMode_Noise MAX abs((V(CAN_H)+V(CAN_L))/2 - 2.5)`

| Metric | Baseline (Unmitigated) | Approach A (Shielded) | Approach B (RC Filter) | Target Requirement |
| :--- | :--- | :--- | :--- | :--- |
| **Peak Common-Mode Noise** | Severe Spikes (FAIL) | ~75 mV deviation | ~70 mV deviation | **< 100 mV** |
| **Differential Amplitude** | Highly erratic | ~2.0 V (Maintained) | ~2.0 V (Maintained) | **> 1.5 V Limits** |
| **Signal Quality** | Eye diagram collapsed | Near-ideal square wave | Damped ringing | **Stable Bit Window** |
| **Expected BER** | Catastrophic failure | Meets $\le 10^{-9}$ | Meets $\le 10^{-9}$ | **$\le 10^{-9}$** |

---

## ⚖️ Engineering Trade-Offs

Physical automotive implementation requires weighing complex constraints:

1. **RC Filtering Limitations:** The 470pF capacitors artificially slow the voltage slew rate, slightly rounding the signal edges. While acceptable for standard 1 Mbps CAN, this RC time constant would collapse the eye diagram of a modern high-speed CAN-FD bus (2-5 Mbps). Furthermore, component tolerance mismatches (e.g., ±1% resistors) can cause uneven discharging, inadvertently converting common-mode noise into differential data corruption.
2. **Shielded Cable Limitations:** Copper shielding adds significant mass and cost to the vehicle harness, conflicting with EV lightweighting goals. Additionally, shielding introduces the risk of ground loops; if grounded across differing chassis potentials, massive DC currents can flow through the shield, generating magnetic noise or thermal damage.

**Final Conclusion:** Shielding provides superior edge-rate preservation for CAN-FD networks, while RC filtering offers an optimal, zero-weight penalty solution for standard 1 Mbps CAN networks.

---

## 🚀 Reproduction Instructions
To reproduce these findings:
1. Install [LTspice](https://www.analog.com/en/design-center/design-tools-and-calculators/ltspice-simulator.html).
2. Clone this repository: 
   ```bash
   git clone [https://github.com/Arka236/CAN-Bus-Signal-Integrity.git](https://github.com/YourUsername/CAN-Bus-Signal-Integrity.git)
