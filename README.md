# Energy-Efficient 32-Point FFT Accelerator on a RISC-V SoC

A custom FFT accelerator integrated into a PicoRV32-based SoC, designed and taken
through the full digital VLSI flow (RTL, synthesis, place and route, post-layout
simulation, sign-off) in a 45 nm CMOS technology. This is the **Energy-Efficient
(EE) variant** of the ET4351 Digital VLSI Systems on Chip course project at TU Delft.

> **Headline result: 9.5x energy reduction over the baseline (24.6 nJ to 2.59 nJ
> per FFT chunk) at 12 MHz, with 0 setup and 0 hold violations after place and route.**

This repository contains the **[final project report](ET4351_VLSI_Project_report.pdf) and selected sign-off
artifacts**. The RTL source code, synthesis and PnR scripts are not included, as
they build on course-provided infrastructure that is not publicly redistributable.

---

## Table of Contents
1. [Overview](#overview)
2. [Results](#results)
3. [Design Methodology](#design-methodology)
4. [Tools and Technology](#tools-and-technology)
5. [Team ](#team)
6. [Acknowledgments](#acknowledgments)

---

## Overview

### What this project does
The host SoC is a PicoRV32 RISC-V microcontroller (`picosoc`) augmented with a
hardware FFT accelerator. The CPU streams real audio samples from external QSPI
flash into the accelerator, which performs a 32-point FFT and writes the complex
spectrum back to SRAM. The CPU then continues with the next audio chunk. Energy
per chunk is the primary metric.

### What "EE" means
Two designs were targeted in parallel by the same team: a **High-Performance (HP)**
variant that pushes the clock to 66 MHz for minimum latency, and an **Energy-Efficient
(EE)** variant that minimizes total energy at the original 12 MHz. This repository
documents the EE design.

### Why it is interesting
The baseline FFT accelerator was bottlenecked by SRAM accesses (about 88 percent
of cycles were memory transfers). The team rebuilt the algorithm and the
microarchitecture from the ground up, replacing SRAM with a local register file,
moving from a 32-point complex Radix-2 FFT to a 16-point pure Radix-4 FFT enabled
by a Real-FFT (RFFT) packing trick, and adding manual clock gating. The result is
roughly an order-of-magnitude improvement in energy with the same area envelope
and the same external interface.

---

## Results

| Metric | Baseline | **EE Design (this work)** | Improvement |
|---|---:|---:|---:|
| Operating frequency | 12 MHz | 12 MHz | unchanged |
| Latency per chunk (cycles) | 732 | **130** | 5.6x |
| Latency per chunk (µs)     | 61.00 | **10.83** | 5.6x |
| Total SoC power (mW)       | 0.626 | **0.239** | 2.6x lower |
| Accelerator dynamic power (mW) | 0.093 | **0.001** | 93x lower |
| **Energy per chunk (nJ)**  | 24.6  | **2.59**  | **9.5x lower** |
| Setup WNS (ns)             | +33.85 | +35.09 | clean |
| Hold WNS (ns)              | +0.032 | +0.041 | clean |
| DRC violations             | 0 | 0 | clean |
| Critical DRV violations    | 0 | 0 | clean |
| VCD activity coverage      | 100% | 100% | clean |
| Core area (µm²)            | 355,693 | 268,310 | within 596.4 x 596.4 µm budget |

All numbers are post place and route, with VCD-annotated power based on a
post-layout hold-corner simulation of one full audio chunk.

The full FFT chunk completes in 130 cycles, broken into the FSM phases:

```
S_INIT --> S_LOAD_DATA --> S_COMPUTE --> S_RECOMBINE --> S_STORE --> S_FINISH
```

LOAD reads only the 32 real input samples (imaginary parts are tied to zero from
reset). COMPUTE runs the two-stage Radix-4 FFT on the 16 packed complex samples.
RECOMBINE unpacks the 16-point result back into a 32-point spectrum using
Hermitian symmetry. STORE writes the spectrum back to memory.

---

## Design Methodology

The EE design was reached through a deliberate sequence of optimizations, each
motivated by analyzing the bottleneck of the previous step.

### 1. Move intermediate state into a register file
Profiling revealed that the baseline accelerator's SRAM is the dominant power
consumer, accounting for over half of the total chip power. This is caused by the
repeated SRAM read/write of intermediate butterfly results. We replaced the
intermediate-result SRAM with a local register file inside the accelerator. SRAM
accesses are now confined to two phases: a single bulk LOAD at the start of a
chunk and a single bulk STORE at the end. Cycle count drops from 732 to 210.

### 2. Constant twiddle strategy with a hardcoded LUT
The baseline either fetched twiddle factors from SRAM or recomputed them
recursively in the butterfly loop. Both options are wasteful when the FFT size is
fixed (N = 32). We embedded all required twiddles directly in the datapath as
fixed Q12 constants. Trivial twiddles (+1, -1, +j, -j) are realized with sign
inversion and real/imaginary swapping, with no multiplier activity at all.

### 3. Skip the imaginary input loading
The audio input is purely real, so 32 of the 64 baseline LOAD cycles were spent
loading zeros. We address-strided the LOAD phase to read only the real parts.
Imaginary parts are tied to zero from reset. LOAD shrinks from 64 to 32 cycles.

### 4. RFFT to enable a pure Radix-4 decomposition
Radix-4 is the obvious next step after constant twiddles, because it halves the
number of stages and exposes more parallelism per butterfly. However, 32 is
2^5, not a power of 4, so a direct Radix-4 decomposition of a 32-point complex
FFT requires a trailing Radix-2 stage (Radix-4-4-2), which the team initially
implemented as an intermediate design. To eliminate the residual Radix-2 stage,
we used the Real FFT (RFFT) trick: a 32-point real input has Hermitian symmetry,
so it can be packed into 16 complex samples and processed as a 16-point FFT.
Since 16 = 4^2, this is a pure two-stage Radix-4. After the FFT, a RECOMBINE
phase unpacks the 16 complex outputs into the 32-point spectrum using:

```
X[k] = (Z[k] + Z*[N/2-k]) / 2  -  j * W_N^k * (Z[k] - Z*[N/2-k]) / 2
```

The total work drops to 8 butterflies (4 per stage) versus 80 in the baseline.
Stage 1 of the Radix-4 uses only trivial twiddles, so it requires no real
multiplications. Stage 2 has 4 unique constant twiddles, hardcoded in a small
LUT.

### 5. Manual integrated clock gating
The previous steps had reduced cycle count and SRAM activity, but accelerator
logic was still toggling whenever the CPU did unrelated work. The team added
**manual clock gating** using TLATNCAX2 ICG cells from the gpdk045 library,
applied to both the FFT datapath and the memory interface. Each ICG latches its
enable signal during the low phase of the clock to prevent enable-glitches from
reaching the gated clock output, then ANDs the latched enable with the clock.

We chose **manual ICG insertion** over Genus's automatic clock gating because
the automatic flow introduced hold violations that could not be resolved within
the design constraints.

### 6. Hold-timing closure
Adding the ICGs and the wide RECOMBINE multiplication path created a non-trivial
hold-fixing problem during PnR. The closure recipe:
- **SDC**: relax `set_max_transition` from 0.20 ns to 0.28 ns to prevent the
  tool from over-buffering high-fanout nets, which would shorten paths and
  create hold violations on short register-file arcs. The relaxation reflects
  the structural reality of the Radix-4 RFFT design (fully combinational
  RECOMBINE multiplication path, high-fanout nets from the parallel Stage 1
  butterfly).
- **PnR (CTS)**: increased `holdTargetSlack` from the baseline 0.1 ns to 0.2 ns,
  driving more aggressive hold fixing during clock tree synthesis and reducing
  the number of violations carried into routing.
- **PnR (post-route)**: a dedicated `optDesign -postRoute -hold` pass with
  `setOptMode -holdTargetSlack 0.05 ns` to close residual violations with
  positive margin rather than stopping at the hold boundary.

Final result: 0 setup violations (WNS +35.1 ns) and 0 hold violations
(WNS +0.041 ns).



## Tools and Technology

| Tool | Role |
|---|---|
| Cadence Genus | RTL synthesis to gate-level netlist |
| Cadence Innovus | Floorplan, placement, CTS, routing, sign-off |
| QuestaSim | Behavioral, structural, and physical simulation |
| Python 3 (NumPy) | Golden FFT reference for verification |
| GCC RISC-V toolchain | Firmware compilation |

| Library | Detail |
|---|---|
| Standard cell library | gsclib045 SVT v4.7 (45 nm) |
| SRAM | saed32sram (single-port macros) |
| ICG cell | TLATNCAX2 (latch-based, glitch-free) |
| Tech file | gpdk045 v6.0 |

| Spec | Value |
|---|---|
| Operating frequency | 12 MHz (83.33 ns) |
| Core area budget | 596.4 µm × 596.4 µm (fixed) |
| FFT size | 32-point real input (16-complex internal) |
| Fixed-point format | Q12 |

---

## Team 

This was a 10-person group project, with 5 members on the EE design and 5 on the
HP design. The EE design described in this repository is the result of the
combined work of the EE sub-team. Per the official project contributions table presented in the report


## Acknowledgments

- TU Delft Microelectronics Department for the ET4351 course infrastructure
- ET4351 course staff at TU Delft for the baseline RTL, the script skeletons,
  and the verification framework
- The PicoRV32 RISC-V core is by Claire Wolf
  (https://github.com/YosysHQ/picorv32)
