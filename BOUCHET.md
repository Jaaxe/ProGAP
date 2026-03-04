# Running ProGAP on Yale Bouchet HPC

This guide documents the **complete workflow for running the ProGAP repository on the Bouchet cluster**, including environment setup, dependency fixes, and large-scale experiment execution.

It also records solutions to issues encountered during setup (PyG build issues, WandB non-TTY errors, etc.).

---

# 1. SSH into Bouchet

```bash
ssh jyx5@bouchet.ycrc.yale.edu
```

---

# 2. Start a persistent terminal session (tmux)

Long jobs and environment setup should be done inside `tmux`.

```bash
tmux new -s progap
```

Detach session:

```
Ctrl+B then D
```

Reconnect later:

```bash
tmux attach -t progap
```

---

# 3. Activate Conda environment

Bouchet uses modules for Python environments.

```bash
module load miniconda
source /apps/software/system/software/miniconda/24.11.3/etc/profile.d/conda.sh
conda activate progap
```

Load CUDA (required for PyTorch builds):

```bash
module load CUDA/12.1.1
```

---

# 4. Navigate to repo

```bash
cd ~/ProGAP
```

Repo structure:

```
ProGAP/
 ├── train.py
 ├── experiments.py
 ├── core/
 ├── config/
 ├── requirements.txt
 └── results.ipynb
```

---

# 5. Set scratch directories (important on Bouchet)

Large data and experiment outputs should go to **scratch**, not `$HOME`.

```bash
export PI_SCRATCH=/nfs/roberts/scratch/pi_ql324/jyx5

export PROGAP_SCRATCH=$PI_SCRATCH/progap
export PROGAP_DATA=$PROGAP_SCRATCH/datasets
export PROGAP_RUNS=$PROGAP_SCRATCH/runs
export PROGAP_LOGS=$PROGAP_SCRATCH/logs
export TMPDIR=$PROGAP_SCRATCH/tmp
```

Create directories:

```bash
mkdir -p $PROGAP_SCRATCH/{datasets,runs,logs,tmp}
```

---

# 6. Install required dependencies

Some dependencies are not automatically installed from `requirements.txt`.

## Install autodp (required by the paper)

```bash
pip install git+https://github.com/yuxiangw/autodp
```

## Install dp_accounting

```bash
pip install dp-accounting
```

---

# 7. Fix PyTorch Geometric dependency (pyg_lib)

Building `pyg_lib` from source can fail due to missing CUDA/CMake configuration.

If `pyg_lib` is missing:

```bash
pip install pyg-lib
```

Verify:

```bash
python -c "import pyg_lib; from pyg_lib.sampler import neighbor_sample"
```

Expected output:

```
(no error)
```

---

# 8. Disable WandB prompts (important for Slurm)

WandB crashes in non-interactive Slurm jobs because it tries to prompt for an API key.

Fix by running in **offline mode**:

```bash
export WANDB_MODE=offline
export WANDB_SILENT=true
```

This prevents the error:

```
UsageError: api_key not configured (no-tty)
```

---

# 9. Quick sanity check run

Before running full experiments, verify training works.

```bash
RUN_DIR=$PROGAP_RUNS/smoke_test
mkdir -p $RUN_DIR

python train.py progap edge \
  --dataset facebook \
  --epsilon 1 \
  --data_dir "$PROGAP_DATA" \
  --output_dir "$RUN_DIR"
```

Expected result:

```
training logs
test/acc ≈ 70-80%
```

---

# 10. Generate paper experiments

The repo automatically generates experiment commands.

```bash
python experiments.py --generate
```

This creates:

```
jobs/experiments.sh
```

Each line is a command like:

```
python train.py progap edge ...
```

Inspect commands:

```bash
sed -n '1,10p' jobs/experiments.sh
```

---

# 11. Run a single experiment

Test the first command:

```bash
cmd=$(sed -n '1p' jobs/experiments.sh)
echo $cmd
eval $cmd
```

---

# 12. Run experiments using Slurm arrays

Running all experiments sequentially would take days.

Instead use **Slurm array jobs**.

Create script:

```bash
nano $PROGAP_SCRATCH/progap_array.sbatch
```

Paste:

```bash
#!/bin/bash
#SBATCH -J progap_paper
#SBATCH -p gpu
#SBATCH --gpus=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=32G
#SBATCH --time=08:00:00
#SBATCH -o /nfs/roberts/scratch/pi_ql324/jyx5/progap/logs/%x_%A_%a.out
#SBATCH -e /nfs/roberts/scratch/pi_ql324/jyx5/progap/logs/%x_%A_%a.err

module load miniconda
source /apps/software/system/software/miniconda/24.11.3/etc/profile.d/conda.sh
conda activate progap
module load CUDA/12.1.1

export PI_SCRATCH=/nfs/roberts/scratch/pi_ql324/jyx5
export PROGAP_SCRATCH=$PI_SCRATCH/progap
export PROGAP_DATA=$PROGAP_SCRATCH/datasets
export PROGAP_RUNS=$PROGAP_SCRATCH/runs
export PROGAP_LOGS=$PROGAP_SCRATCH/logs

export WANDB_MODE=offline
export WANDB_SILENT=true

cd ~/ProGAP

cmd=$(sed -n "${SLURM_ARRAY_TASK_ID}p" jobs/experiments.sh)

echo "Running task ${SLURM_ARRAY_TASK_ID}"
echo $cmd

eval $cmd
```

---

# 13. Submit the array job

Count experiments:

```bash
wc -l jobs/experiments.sh
```

Example:

```
2840 experiments
```

Submit with limited concurrency:

```bash
sbatch --array=1-2840%10 $PROGAP_SCRATCH/progap_array.sbatch
```

`%10` limits to **10 GPUs running simultaneously**.

---

# 14. Monitor jobs

Check queue:

```bash
squeue -u $USER
```

Check logs:

```bash
ls -lt $PROGAP_LOGS
```

Follow a job:

```bash
tail -f $PROGAP_LOGS/progap_paper_<jobid>_1.out
```

---

# 15. Cancel jobs if needed

Cancel a job:

```bash
scancel JOBID
```

Cancel all jobs:

```bash
scancel -u $USER
```

---

# 16. Generate paper plots

After experiments finish:

Open:

```
results.ipynb
```

Run all cells to reproduce paper figures.

---

# Summary Workflow

```
SSH into Bouchet
↓
tmux session
↓
activate conda env
↓
set scratch directories
↓
generate experiments
↓
test one experiment
↓
submit Slurm array
↓
monitor jobs
↓
run results notebook
```

---

# Notes

### Why tmux?
Prevents environment loss if SSH disconnects.

### Why scratch?
Home directories have limited space.

### Why WandB offline?
Slurm jobs have no interactive terminal.

### Why Slurm arrays?
Allows running thousands of experiments efficiently.

---

# Quick Commands Cheat Sheet

```
ssh jyx5@bouchet.ycrc.yale.edu
tmux new -s progap
conda activate progap
cd ~/ProGAP
python experiments.py --generate
sbatch --array=1-2840%10 progap_array.sbatch
squeue -u $USER
```

---
