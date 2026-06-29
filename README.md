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
- Siemens Questa or ModelSim
- Xilinx Vivado with support for `xc7k160tffg676-2`
The Kintex-7 part `xc7k160tffg676-2` is used by Vivado for synthesis, implementation, and route as the required Kintex-7 XC7K325T is not available.
 
Install GNU Make:
 
```powershell
choco install make
```
 
Typical PATH entries:
 
```text
C:\Xilinx\Vivado\2023.2\bin
C:\questasim64_2023.2\win64
```
 
Set PATH for the current PowerShell session:
 
```powershell
$env:Path += ";C:\Xilinx\Vivado\2023.2\bin"
$env:Path += ";C:\questasim64_2023.2\win64"
```
 
Open the Questa Sim license server/session using PuTTY before running simulation.
 
## Install Python Dependencies
 
From the project root:
 
```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu
pip install pillow
pip install numpy
pip install sentence-transformers
pip install transformers huggingface-hub scikit-learn scipy tqdm
```
 
## Run RTL Simulation
 
Run the default simulation:
 
```powershell
make sim
```
 
Run simulation with a specific matrix size (N = 1, 2, 3, or 4):
 
```powershell
make sim N=3
```
 
Run simulation through the Python image flow. The `--task` argument accepts a task number from 1-14:
 
```powershell
python sw/infer.py --image sw/test.jpg --task 1
```
 
Alternatively, use `--task_nl` to specify one of the 14 COCO tasks by name instead of by number:
 
```powershell
python sw/infer.py --image sw/test.jpg --task_nl "Sit comfortably"
```
 
Run matrix verification through Python. The `--n` argument accepts a matrix order number from 1-4:
 
```powershell
python sw/infer.py --matmul --n 3
```
 
> Note: If `python` is not recognized or fails to run the commands above, try `py` instead of `python`.
 
Simulation outputs include:
 
- `result.mem`
- `matrix_rtl.mem`
- `rtl_metrics.txt`
- `vsim.wlf`
- `outputs/reports/sim/`
## Run Synthesis And Implementation
 
Run synthesis:
 
```powershell
make synth
```
 
Run implementation:
 
```powershell
make impl
```
 
Run route:
 
```powershell
make route
```
 
Run simulation, synthesis, implementation, and route:
 
```powershell
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
 
```powershell
make clean
```
 
