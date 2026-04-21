# Running RFDiffusion3 (RFD3) via Foundry

While working with RFDiffusion3 (RFD3) through Foundry to redesign OATP1B1, several debugging steps were required to properly configure and execute the pipeline. Below is a structured summary of key lessons, setup steps, and execution workflow.

---

## Key Lessons & Debugging Insights

### 1. RFD3 is NOT a Python Importable Package
Attempts to import RFD3 directly:
```python
from rfdiffusion3.applications import rfdiffusion3
from foundry.applications import rfdiffusion3
```
Resulted in:
```
ModuleNotFoundError
```

✅ **Solution:**  
RFD3 must be run via the CLI:
```bash
rfd3 design
```

---

### 2. Correct Installation of Foundry

Incorrect attempts:
```bash
pip install -e src
pip install -e models/rfd3
```

These failed because the repo is not structured as an installable package.

✅ **Correct Installation:**
```bash
pip install "rc-foundry[rfd3]"
```

Verify:
```bash
python -m pip show rc-foundry
```

---

### 3. Disk Quota & Environment Setup

Due to home directory limits, a Conda environment was created in scratch space:

```bash
cd /mnt/gs21/scratch/cordahlr

mkdir -p conda_envs

conda create -p /mnt/gs21/scratch/cordahlr/conda_envs/rfd3 python=3.12 -y
conda activate /mnt/gs21/scratch/cordahlr/conda_envs/rfd3
```

---

### 4. Input Requirements (Critical)

RFD3 does **NOT** accept:
- `.pdb` files directly
- `.yaml` files as inputs

Errors encountered:
```
ValueError: Expected args to be a dictionary
ValidationError: Either 'input' or 'contig' / 'length' must be provided
```

✅ **Solution:**  
Inputs must be provided as a **JSON dictionary**

---

### 5. Correct JSON Format

❌ Incorrect (list format):
```
AttributeError: 'list' object has no attribute 'items'
```

✅ Correct format:
```json
{
  "oatp1b1_example": {
    "input_pdb": "/mnt/gs21/scratch/cordahlr/rfd3/inputs/oatp1b1/oatp1b1_e3s_rfd.pdb",
    "mode": "partial_diffusion",
    "fixed_backbone": true,
    "redesign_residues": [352, 356, 386, 422, 541, 544, 625],
    "ligand": {
      "present": true
    }
  }
}
```

---

### 6. Hydra Config vs Input Separation

Errors:
```
ConfigCompositionException
TypeError: unexpected keyword argument 'inference'
```

❗ Cause: Mixing runtime config with input JSON

✅ **Fix:**
- JSON → input data
- CLI → runtime configuration

---

### 7. Checkpoint Handling

❌ Incorrect (Python):
```python
RFD3InferenceEngine(checkpoint="...")
```

Error:
```
TypeError: unexpected keyword argument 'checkpoint'
```

✅ **Correct CLI usage:**
```bash
rfd3 design \
  +checkpoint=/mnt/gs21/scratch/cordahlr/foundry_ckpts/rfd3_latest.ckpt \
  inputs=/mnt/gs21/scratch/cordahlr/rfd3/inputs/oatp1b1/oatp1b1.json \
  out_dir=/mnt/gs21/scratch/cordahlr/rfd3/outputs \
  prevalidate_inputs=True \
  skip_existing=False
```

---

### 8. Environment Variables

```bash
export HYDRA_FULL_ERROR=1
export CUDA_LAUNCH_BLOCKING=0
export FOUNDRY_CHECKPOINT_DIRS=/mnt/gs21/scratch/cordahlr/foundry_ckpts
```

---

## SLURM Job Script (GPU Execution)

```bash
#!/bin/bash --login
#SBATCH --job-name=rfd3_oatp1b1
#SBATCH --output=logs/slurm/%x_%j.out
#SBATCH --error=logs/slurm/%x_%j.err
#SBATCH --time=08:00:00
#SBATCH --cpus-per-task=8
#SBATCH --mem=48G
#SBATCH --gres=gpu:1

set -euo pipefail

module purge
module load Miniforge3

source "$(conda info --base)/etc/profile.d/conda.sh"
conda activate /mnt/gs21/scratch/cordahlr/conda_envs/rfd3

export HYDRA_FULL_ERROR=1
export CUDA_LAUNCH_BLOCKING=0
export FOUNDRY_CHECKPOINT_DIRS=/mnt/gs21/scratch/cordahlr/foundry_ckpts

cd /mnt/gs21/scratch/cordahlr/rfd3
mkdir -p logs/slurm outputs

echo "Starting RFD3 run..."
nvidia-smi

rfd3 design \
  +checkpoint=/mnt/gs21/scratch/cordahlr/foundry_ckpts/rfd3_latest.ckpt \
  out_dir=outputs/run_0 \
  inputs=inputs/oatp1b1/oatp1b1.json \
  prevalidate_inputs=True \
  low_memory_mode=True \
  diffusion_batch_size=1 \
  n_batches=20 \
  skip_existing=False

echo "Job complete."
```

---

## Current Status

- ✅ Environment configured correctly  
- ✅ CLI execution working  
- ✅ Checkpoint successfully loaded  

❗ **Remaining Issue:**
```
ValidationError: Unsupported file type
```

### Likely Causes:
- Incorrect file path inside JSON  
- Unsupported file extension  
- Improper JSON structure  

---

## Key Takeaways

- Always use CLI: `rfd3 design`  
- Always use **JSON** for inputs  
- Never pass checkpoint via Python  
- Use `+checkpoint=` with Hydra  
- Keep environments in scratch space  
- Validate JSON before execution
