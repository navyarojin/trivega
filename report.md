# TriVega Accelerator Project Report

## Introduction

The project implements a system that takes an image and a task description, then identifies the most appropriate object in the scene based on task relevance rather than generic object detection alone as a software-assisted, FPGA-oriented pipeline: software prepares COCO-style object candidates and task encodings, while RTL performs fixed-point scoring, affordance matching, fusion, and best-object selection through a custom AXI4-controlled accelerator.

## 1. Block Diagram of the Design

The system addresses task-aware object selection problem: given an image and one of the 14 reference tasks, the system selects the most appropriate object in the scene. The design uses software to prepare COCO object candidates and task information, then uses a custom fixed-point RTL accelerator to score and select the best object.

```mermaid
flowchart LR
    IMG["Input image<br/>COCO-style scene"] --> SW["Software front end<br/>object detection + task encoding"]
    TASK["Task input<br/>14 reference tasks / text prompt"] --> SW

    SW -->|"object records, boxes,<br/>task word, pixels"| DDR["DDR memory"]
    HOST["VEGA / host processor"] -->|"custom AXI4 register/control"| REGS["axi4_regs<br/>control/status/counters"]
    REGS --> CORE["vega_core<br/>top controller"]
    CORE --> DMA["axi4_dma<br/>mode FSM + datapath control"]

    DMA -->|"AXI4 full read/write"| DDR
    DMA --> RD["dma_rd<br/>burst read engine"]
    DMA --> WR["dma_wr<br/>burst write engine"]

    DMA --> BUF["On-chip buffers<br/>object data, boxes, scene cache, scores"]
    BUF --> PROJ["embed_proj<br/>fixed-point projection"]
    PROJ --> COS["cos_sim<br/>task similarity"]
    COS --> AFF["aff_score<br/>affordance match"]
    AFF --> FUSE["score_fuse<br/>weighted score fusion"]
    FUSE --> TOP1["top1_sel<br/>best object selector"]
    TOP1 --> WR
    WR -->|"best object index / scores"| DDR

    DMA --> MM["Matrix mode<br/>N=1..4 fixed-point matmul"]
    MM --> PROJ
    CORE --> IRQ["interrupt"]
```

The register/control interface is custom AXI4-style, not AXI4-Lite. The slave-side register block exposes AXI4 full-style channels and sideband signals, while the accelerator uses an AXI4 full master interface for DDR access.

## 2. Simulation Setup Block Diagram

```mermaid
flowchart LR
    MAKE["Makefile<br/>make sim / make sim N=3"] --> SIM["Questa/ModelSim"]
    PY["sw/infer.py<br/>image + matrix driver"] --> MEMFILES["objects.mem, task.mem,<br/>matrix_a.mem, matrix_b.mem"]
    MEMFILES --> TB["tb_vega.sv<br/>self-checking testbench"]
    SIM --> TB

    TB -->|"custom AXI4 CPU writes/reads"| DUT["DUT: vega_core"]
    DUT -->|"AXI4 full master"| DDRMODEL["ddr_mem.sv<br/>behavioral DDR model"]
    DDRMODEL --> DUT

    TB --> OUT["result.mem<br/>matrix_rtl.mem<br/>rtl_metrics.txt<br/>vsim.wlf"]
    PY --> REF["Python reference<br/>matrix_ref.mem"]
    OUT --> CMP["Result comparison<br/>latency + acceleration"]
    REF --> CMP
```

Simulation is driven by `tb/tb_vega.sv`. The testbench creates default image/task memory files if absent, configures the accelerator through the custom AXI4 register interface, waits for completion, checks the image-mode result, and then runs matrix mode. Matrix mode writes `rtl_metrics.txt` with cycle count, latency, and clock frequency.

## 3. Summary of RTL

| RTL file | Summary |
| --- | --- |
| `rtl/top.sv` | Synthesis wrapper exposing slave AXI, master AXI, clock/reset, and interrupt. |
| `rtl/vega_core.sv` | Main controller. Validates configuration, controls `IDLE`, `ARMED`, `RUNNING`, `DONE`, and `ERR` states, tracks cycle/DMA counters, and connects registers to the DMA datapath. |
| `rtl/axi4_regs.sv` | Custom AXI4-style register/control bank for start/reset, status, base addresses, mode, object count, matrix size, cycle count, processed objects, DMA reads, and DMA writes. |
| `rtl/axi4_dma.sv` | Main datapath FSM. Supports image scoring and matrix multiplication; sequences DDR reads, ROI packing, feature projection, similarity, affordance scoring, score fusion, top-1 selection, and DDR writes. |
| `rtl/dma_rd.sv` | AXI4 burst read engine for input, task, and matrix data. |
| `rtl/dma_wr.sv` | AXI4 burst write engine for scores, best index, and matrix output. |
| `rtl/embed_proj.sv` | Fixed-point multiply-accumulate projection stage reused by image and matrix modes. |
| `rtl/cos_sim.sv` | Fixed-point cosine-style similarity computation. |
| `rtl/aff_score.sv` | Object affordance lookup and task affordance matching. |
| `rtl/score_fuse.sv` | Weighted fusion of detection, similarity, and affordance scores. |
| `rtl/top1_sel.sv` | Selects the best-scoring object. |

Key design parameters:

| Item | Value |
| --- | --- |
| Data format | 16-bit fixed point, `FRAC=12` |
| AXI data width | 64 bits |
| Maximum objects | 16 in RTL, 13 in default testbench |
| Maximum burst length | 16 beats |
| Image input limit | 816 x 648 pixels |
| Scene cache | 196,608 pixels |
| ROI samples per object | 256 |
| Matrix mode | `N=1..4` |
| Simulation clock | 50 MHz, 20 ns period |
| Contest target | VEGA processor on Genesys-2 FPGA board |
| Current Vivado script target | Kintex-7 `xc7k160tffg676-2` substitute part |

Main register map:

| Offset | Register |
| --- | --- |
| `0x00` | Control, start, soft reset |
| `0x08` | Status: done, error, state, object count |
| `0x10` | Input base address |
| `0x18` | Task or matrix-B base address |
| `0x20` | Output base address |
| `0x28` | Number of objects or matrix dimension copy |
| `0x30` | Cycle counter |
| `0x34` | Objects processed |
| `0x38` | DMA read beat counter |
| `0x3C` | DMA write beat counter |
| `0x40` | Mode: `0=image`, `1=matmul` |
| `0x48` | Matrix dimension `N` |

## 4. Simulation Results

Simulation was run from the project root using:

```powershell
make sim
```

The Questa/ModelSim run completed successfully with zero compile warnings, zero compile errors, zero simulation warnings, and zero simulation errors.

| Test | Observed result |
| --- | --- |
| Image mode | Passed. `result.mem` contains best object index `0`, valid for the default `NUM_OBJECTS=13` testbench stimulus. |
| Matrix mode | Passed for `N=3`. RTL generated `matrix_rtl.mem` and `rtl_metrics.txt`. |
| Overall result | `TB PASS` |
| RTL/software comparison | Matrix RTL output matches the Python reference bit-for-bit. |

Generated result files:

| File | Purpose |
| --- | --- |
| `result.mem` | Best object index from image mode. |
| `matrix_rtl.mem` | Matrix output produced by RTL. |
| `matrix_ref.mem` | Matrix output produced by Python reference flow. |
| `rtl_metrics.txt` | Contains cycle count, latency, and frequency. |
| `vsim.wlf` | Questa/ModelSim waveform database. |

Measured matrix-mode metrics:

| Metric | Value |
| --- | --- |
| Matrix size | `N=3` |
| RTL cycles | 223 |
| RTL latency | 4.460 us |
| RTL frequency | 50 MHz |
| Matrix output words | 9 |

## 5. Current Status

The current project is functionally verified in RTL simulation. Remaining contest-facing work is board integration with VEGA/Genesys-2 and end-to-end image inference.

## 6. Acceleration Achieved

The acceleration measurement uses matrix mode as the direct hardware/software comparison path. The software flow computes:

```text
hardware_acceleration = python_time_us / rtl_latency_us
rtl_latency_us = rtl_cycles / 50.0
```

Measured RTL metric:

```text
cycles=223
latency_us=4.460000
freq_mhz=50.000000
```

Hardware/software comparison for `N=3`:

```text
N=3
python_time_us=23.400
rtl_cycles=223
rtl_latency_us=4.460
hardware_acceleration=5.246638x
PASS: RTL output matches software
```

| Metric | Value |
| --- | --- |
| RTL clock | 50 MHz |
| RTL cycles | 223 |
| RTL latency | 4.460 us |
| Software reference time | 23.400 us |
| Acceleration achieved | 5.246638x, approximately 5.25x |

```mermaid
xychart-beta
    title "N=3 Latency Comparison"
    x-axis ["RTL", "Python"]
    y-axis "Latency (us)" 0 --> 25
    bar [4.46, 23.40]
```
