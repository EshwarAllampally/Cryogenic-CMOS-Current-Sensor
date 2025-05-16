# Cryogenic CMOS Current Integrator & CDS Amplifier


## Overview

This project explores a **Cryogenic CMOS Current Comparator** inspired by a real-world spin qubit readout circuit. Although our implementation was done at room temperature, we:

* Replicated the core **Current Integrator** and **CDS Amplifier** blocks from literature
* Used **Cadence Virtuoso** for schematic design and simulations
* Designed a **custom Verilog-A Op-Amp** to drive both analog blocks
* Achieved current sensing in the **<1nA** regime

## What is CDS and Who Invented It?

**Correlated Double Sampling (CDS)** was introduced in the 1960s by **Willard Boyle and George Smith**—yup, the CCD inventors! It was developed to tackle **low-frequency noise and offset voltage** issues in imaging sensors.

CDS takes two samples:

1. Before the actual signal arrives (baseline)
2. After the signal is captured

Subtracting these removes offset and **1/f noise**, which is vital for ultra-low current readouts like in spin qubits.

## Figures of Merit (FOM)

| **Metric**                | **Description**                      | **Our Design**          |
| ------------------------- | ------------------------------------ | ----------------------- |
| Input Current Sensitivity | Minimum detectable current           | < 1 nA                  |
| Integration Window        | Time period for charge accumulation  | 1–1000 μs (tunable)     |
| Opamp Gain                | DC gain of the Verilog-A opamp       | \~80 dB                 |
| Area                      | Estimated block-level footprint      | \~0.045 mm²             |
| Power Supply              | Operating voltage                    | 1.2 V                   |
| Readout Fidelity          | Success probability for correct read | \~99.9% (ref benchmark) |
| CDS Noise Suppression     | Reduction in flicker/thermal noise   | \~20 dB (sim est.)      |


## Traditional Approaches vs Our Design

| **Method**               | **Problems**                                   | **Why Ours is Better**                 |
| ------------------------ | ---------------------------------------------- | -------------------------------------- |
| Direct ADC Sampling      | Needs high-res ADC, suffers from low SNR       | Integrator boosts the signal           |
| Current Mirror Readout   | Mismatch issues at low current                 | CDS suppresses offsets                 |
| Transimpedance Amplifier | Challenging at <1nA, requires high gain opamps | Simple, efficient integration approach |

### Why Integrator + CDS?

* Integrator = temporal gain (I \* time = voltage)
* CDS = wipes out slow-varying offsets and 1/f noise
* Combined = Clean, amplified, low-noise signal that’s ADC-friendly

## Verilog-A Opamp Code Snippet

Here's a minimal version of the behavioral opamp we used in Verilog-A:

```verilog
`include "constants.vams"
`include "disciplines.vams"

module opamp(p, n, out);

  inout p, n, out;
  electrical p, n, out;

  parameter real dc_gain = 80;         // dB
  parameter real ugbw = 10e6;          // Hz
  parameter real vdd = 1.8;            // VDD
  parameter real vss = 0.0;            // VSS
  parameter real Vout_offset = 0.2;    // Output-referred offset voltage (default 0V)

  real gain_lin, pole_rad, cap_eq;
  real vout_ideal;

  analog begin

    @(initial_step) begin
        V(out) <+ 1.3 + Vout_offset;   // Initial output voltage with offset
    end

    gain_lin = pow(10, dc_gain / 20.0);
    pole_rad = ugbw * `M_TWO_PI;
    cap_eq = gain_lin / pole_rad;

    // Ideal opamp output without offset
    vout_ideal = gain_lin * (V(p) - V(n));

    // Inject offset into output equation
    I(out) <+ cap_eq * ddt(V(out)) + V(out) - (vout_ideal + Vout_offset);

  end

endmodule
```

> We used this for fast convergence and ideal behavior in early stages.


## Simulated Output Waveforms

We validated our design with realistic test inputs. Highlights:

* **Integrator Output (Ramp Current Input)**

![Clear linear ramp response, slope proportional to input current](https://github.com/EshwarAllampally/Cryogenic-CMOS-Current-Sensor/blob/main/Assets/media/1_integrator_ramp_out.png)

* **CDS Output**

![Offset and low-frequency noise significantly suppressed](https://github.com/EshwarAllampally/Cryogenic-CMOS-Current-Sensor/blob/main/Assets/media/2_CDS_Amp_out_1.png)

* **Comparator Output**

![Clear digital HIGH/LOW levels corresponding to current thresholds](https://github.com/EshwarAllampally/Cryogenic-CMOS-Current-Sensor/blob/main/Assets/media/3_comparator_out.png)

### Example: Comparator Transient

```text
Input: 50 pA | Tint: 180 µs
Result: Clean transition across Vref
```

These matched the expected waveforms from the reference paper.

---

## 📐 Area / Footprint Estimation

Our guesstimate (based on typical 65nm CMOS layout metrics):

* **Integrator + CDS + Opamp** ≈ **0.045 mm²**

  * 60% by capacitors and analog switches
  * 40% by bias circuits and amp transistors

> We skipped layout due to time, but the block-level design gives a realistic ballpark.

---

## 🔭 Next Steps (Where This Could Go)

1. **Use Cryo Models**: Redo sims using actual cryogenic PDKs (like Intel/SkyWater)
2. **Full Layout + DRC/LVS**: Tape-out ready block for MPW shuttles
3. **Noise & Mismatch Analysis**: Monte Carlo + PVT variation sweeps
4. **Digital Integration**: Hook up to a backend readout chain or controller
5. **Packaging + Measurement**: Integrate with real quantum chips (SET or QPC)

---

## Authors

* **Eshwar (MT2024504)** 
* **Manoj (MT2024543)**

---

## References

* IEEE Paper: *A Cryogenic CMOS Current Integrator and CDS for Spin Qubit Readout* \[Fuketa et al., 2023]
* Boyle & Smith: CDS invention in CCDs (Bell Labs)

## Final Word

This was a dive into the nitty-gritty of **precision analog design**, with a splash of **quantum hardware relevance**. The integrator + CDS combo is a practical analog front-end that scales with CMOS — and could be the unsung hero of low-temp quantum readout.

---
