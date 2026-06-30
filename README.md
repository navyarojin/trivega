# TriVega Accelerator
 
## Project Files
 
```text
constraints/
  constraints.xdc        Vivado timing constraints
 
models/
  model.pt               Python Model checkpoint
 
rtl/
  top.sv                 Synthesis wrapper
  vega_core.sv           Main accelerator core
  axi4_regs.sv           AXI4 register/control interface
  axi4_dma.sv            DMA controller
  dma_rd.sv              AXI read engine
  dma_wr.sv              AXI write engine
  embed_proj.sv          Feature projection datapath
  cos_sim.sv             Similarity computation
  aff_score.sv           Affordance score logic
  score_fuse.sv          Score fusion logic
  top1_sel.sv            Best-result selector
 
tb/
  tb_vega.sv             Main testbench
  ddr_mem.sv             DDR memory model
 
scripts/
  sim/modelsim.ini       Questa Sim library setup
  sim/wave.do            Waveform setup
  vivado/sim.tcl         Vivado/XSim simulation script
  vivado/run.tcl         Vivado run script
  vivado.cmd             Vivado launch file
 
sw/
  vega.py                Task/object model logic
  infer.py               Python flow to generate inputs, run RTL, and render output
  test.jpg               Sample input image
 
outputs/
  reports/               Vivado and simulation reports
  logs/                  Vivado logs and journals
  vivado_project/        Generated Vivado project
 
Makefile                 Run commands
README.md                Instructions to run
Report.pdf               Stage 2B report 
AXIinterface.pdf         AXI interface details
```
 
## Required Software
 
- Python 3.10+
- GNU Make
- Siemens QuestaSim
- Xilinx Vivado with support for `xc7k160tffg676-2`
The Kintex-7 part `xc7k160tffg676-2` is used by Vivado for synthesis, implementation, and route as the required Kintex-7 XC7K325T is not available.

This project can be built and run on both Windows and Linux. Vivado and QuestaSim each ship native Windows and Linux installers; install whichever matches your OS and adjust the PATH entries below accordingly.
 
Install GNU Make:

**Windows (PowerShell):**

```text
choco install make
```

**Linux (Debian/Ubuntu):**

```bash
sudo apt-get update
sudo apt-get install make
```

Typical PATH entries:

**Windows:**

```text
C:\Xilinx\Vivado\2023.2\bin
C:\questasim64_2023.2\win64
```

**Linux:**

```text
/opt/Xilinx/Vivado/2023.2/bin
/opt/questasim64_2023.2/linux_x86_64
```

Set PATH for the current session:

**Windows (PowerShell):**

```text
$env:Path += ";C:\Xilinx\Vivado\2023.2\bin"
$env:Path += ";C:\questasim64_2023.2\win64"
```

**Linux (bash):**

```bash
export PATH="$PATH:/opt/Xilinx/Vivado/2023.2/bin"
export PATH="$PATH:/opt/questasim64_2023.2/linux_x86_64"
```

Open the Questa Sim license server/session using PuTTY (Windows) or `ssh` (Linux) before running simulation.

## Install Python Dependencies

From the project root:

**Windows (PowerShell):**

```text
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu
pip install pillow
pip install numpy
pip install sentence-transformers
pip install transformers huggingface-hub scikit-learn scipy tqdm
```

**Linux (bash):**

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu
pip install pillow
pip install numpy
pip install sentence-transformers
pip install transformers huggingface-hub scikit-learn scipy tqdm
```

> Note: on Linux, use `python3` instead of `python` if `python` is not recognized (e.g. `python3 -m venv .venv`).
 
## Run RTL Simulation

The intended RTL simulation flow uses Siemens QuestaSim. Vivado/XSim is also
provided as a temporary waveform-viewing fallback when the Questa license is not
available.
 
The `make` and `python` commands below are identical on Windows (PowerShell) and Linux (bash); only the venv activation and PATH setup above differ between platforms.

Run the default simulation:
 
```text
make sim
```
 
Run simulation with a specific matrix size (N = 1, 2, 3, or 4):
 
```text
make sim N=3
```

Run the Vivado/XSim equivalent in GUI mode:

```text
make viv_sim
```

Run the Vivado/XSim equivalent with a specific matrix size:

```text
make viv_sim N=3
```

Run the Vivado/XSim equivalent in batch mode:

```text
make viv_sim_batch
```

The Vivado/XSim flow compiles the same RTL and testbench files as `make sim`,
passes the same `MATRIX_N` setting, and recreates the waveform groups from
`scripts/sim/wave.do`.
 
Run simulation through the Python image flow. The `--task` argument accepts a task number from 1-14:
 
```text
python sw/infer.py --image sw/test.jpg --task 1
```
 
Alternatively, use `--task_nl` to specify one of the 14 COCO tasks by name instead of by number:
 
```text
python sw/infer.py --image sw/test.jpg --task_nl "Sit comfortably"
```
 
Run matrix verification through Python. The `--n` argument accepts a matrix order number from 1-4:
 
```text
python sw/infer.py --matmul --n 3
```
 
> Note: If `python` is not recognized or fails to run the commands above, try `py` instead of `python` on Windows, or `python3` on Linux.
 
Simulation outputs include:
 
- `result.mem`
- `matrix_rtl.mem`
- `rtl_metrics.txt`
- `vsim.wlf`
- `outputs/reports/sim/`
- `outputs/vivado_sim/` when using `make viv_sim` or `make viv_sim_batch`

## Run Synthesis And Implementation
 
Run synthesis:
 
```text
make synth
```
 
Run implementation:
 
```text
make impl
```
 
Run route:
 
```text
make route
```
 
Run simulation, synthesis, implementation, and route:
 
```text
make all
```
 
Vivado outputs are written under:
 
```text
outputs/vivado_project/
outputs/reports/
outputs/logs/
```
 
## Clean
 
Remove generated outputs:
 
```text
make clean
```
 
