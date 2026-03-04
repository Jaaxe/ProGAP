# Running ProGAP on Yale's Bouchet Cluster

This guide documents the full workflow for running the **ProGAP** repository on Yale's **Bouchet HPC cluster**, including solutions to environment and dependency issues encountered during setup.

The goals of this setup are:

- **Zero changes to the original repo code**
- **Reproducible environment**
- **Proper HPC usage**
- **All datasets/results stored in scratch**

---

# 1. SSH into Bouchet

From your local machine:

```bash
ssh jyx5@bouchet.ycrc.yale.edu
```

It is highly recommended to use **tmux** so your session survives SSH disconnects.

Start tmux:

```bash
tmux new -s progap
```

Detach from tmux:

```
Ctrl + b
d
```

Reconnect later:

```bash
tmux attach -t progap
```

---

# 2. Repository Location

Clone the repository into your **home directory**:

```bash
cd ~
git clone <repo_url> ProGAP
cd ~/ProGAP
```

Home directory is appropriate for **code**, since it is backed up.

---

# 3. Standard Scratch Directory

Datasets, outputs, logs, and temporary files should go to scratch storage.

Set environment variables:

```bash
export PI_SCRATCH=/nfs/roberts/scratch/pi_ql324/jyx5
export PROGAP_SCRATCH=$PI_SCRATCH/progap

export PROGAP_DATA=$PROGAP_SCRATCH/datasets
export PROGAP_RUNS=$PROGAP_SCRATCH/runs
export PROGAP_LOGS=$PROGAP_SCRATCH/logs
export TMPDIR=$PROGAP_SCRATCH/tmp

export PIP_DISABLE_PIP_VERSION_CHECK=1
```

Create the directories:

```bash
mkdir -p $PROGAP_SCRATCH/{datasets,runs,logs,tmp,env,commands}
```

---

# 4. Request an Interactive GPU Node

For debugging and development use the **gpu_devel partition**.

Example allocation:

```bash
salloc -p gpu_devel --gpus=1 --cpus-per-task=2 --mem=10G --time=01:00:00
```

Once allocated your prompt will change to something like:

```
jyx5@a1122u02n01.bouchet
```

You are now on a **compute node with GPU access**.

---

# 5. Activate Conda Environment

Load miniconda and activate the environment:

```bash
module load miniconda
source /apps/software/system/software/miniconda/24.11.3/etc/profile.d/conda.sh
conda activate progap
```

If the environment does not exist yet:

```bash
conda create -n progap python=3.10
conda activate progap
```

---

# 6. Load CUDA Toolkit

Some PyTorch Geometric components require the CUDA toolkit.

```bash
module load CUDA/12.1.1
```

This was necessary when compiling `pyg-lib`.

---

# 7. Install Python Dependencies

Install repository dependencies:

```bash
python -m pip install -r requirements.txt
```

However, two packages required special handling.

---

# 8. Install `dp_accounting`

During training we encountered:

```
ModuleNotFoundError: No module named 'dp_accounting'
```

Install with:

```bash
python -m pip install dp-accounting
```

Verify installation:

```bash
python -c "import dp_accounting; print('dp_accounting ok')"
```

---

# 9. Install `pyg_lib` (Compile From Source)

Standard installation failed because prebuilt wheels were unavailable for the Bouchet environment.

First load CUDA:

```bash
module load CUDA/12.1.1
```

Then build from source:

```bash
export MAX_JOBS=2
export CMAKE_ARGS="-DBUILD_TEST=OFF -DBUILD_BENCHMARK=OFF"

python -m pip install \
  --no-cache-dir \
  --no-build-isolation \
  "pyg_lib @ git+https://github.com/pyg-team/pyg-lib.git"
```

This resolves errors such as:

```
Could NOT find CUDA
cmake returned non-zero exit status
Failed building wheel for pyg_lib
```

Verify installation:

```bash
python -c "import pyg_lib; from pyg_lib.sampler import neighbor_sample; print('pyg_lib sampler ok')"
```

---

# 10. Verify the Environment

Run sanity checks:

```bash
python -c "import torch; print('torch', torch.__version__)"
python -c "import torch_geometric; print('pyg', torch_geometric.__version__)"
python -c "import pyg_lib; print('pyg_lib ok')"
python -c "import dp_accounting; print('dp_accounting ok')"
```

---

# 11. Run a Smoke Test

Navigate to the repository:

```bash
cd ~/ProGAP
```

Create a run directory:

```bash
RUN_TAG=smoke_$(date +%Y%m%d_%H%M%S)
RUN_DIR=$PROGAP_RUNS/$RUN_TAG
mkdir -p "$RUN_DIR"
```

Run training:

```bash
python train.py progap edge \
  --dataset facebook \
  --epsilon 1 \
  --data_dir "$PROGAP_DATA" \
  --output_dir "$RUN_DIR"
```

Expected runtime is **~2 seconds**.

Example output:

```
test/acc   76.14
Total running time: ~2 seconds
Max GPU memory used = 0.16 GB
```

---

# 12. Running Jobs with Slurm

Create a batch script in scratch:

```bash
cat > $PROGAP_SCRATCH/run_progap_one.sbatch <<'EOF'
#!/bin/bash
#SBATCH -J progap_one
#SBATCH -p gpu
#SBATCH --gpus=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=32G
#SBATCH --time=02:00:00
#SBATCH -o /nfs/roberts/scratch/pi_ql324/jyx5/progap/logs/progap_one_%j.out
#SBATCH -e /nfs/roberts/scratch/pi_ql324/jyx5/progap/logs/progap_one_%j.err

set -euo pipefail

module load miniconda
source /apps/software/system/software/miniconda/24.11.3/etc/profile.d/conda.sh
conda activate progap

module load CUDA/12.1.1 || true

export PI_SCRATCH=/nfs/roberts/scratch/pi_ql324/jyx5
export PROGAP_SCRATCH=$PI_SCRATCH/progap
export PROGAP_DATA=$PROGAP_SCRATCH/datasets
export PROGAP_RUNS=$PROGAP_SCRATCH/runs
export PROGAP_LOGS=$PROGAP_SCRATCH/logs
export TMPDIR=$PROGAP_SCRATCH/tmp

RUN_TAG=sbatch_$(date +%Y%m%d_%H%M%S)
RUN_DIR=$PROGAP_RUNS/$RUN_TAG
mkdir -p "$RUN_DIR"

cd /home/jyx5/ProGAP

python train.py progap edge \
  --dataset facebook \
  --epsilon 1 \
  --data_dir "$PROGAP_DATA" \
  --output_dir "$RUN_DIR"
EOF
```

Submit the job:

```bash
sbatch $PROGAP_SCRATCH/run_progap_one.sbatch
```

---

# 13. Monitor Jobs

View running jobs:

```bash
squeue -u $USER
```

Inspect job details:

```bash
scontrol show job <JOB_ID>
```

Watch logs:

```bash
tail -f $PROGAP_LOGS/progap_one_<JOB_ID>.out
```

---

# 14. Pending Jobs Due to Priority

If a job shows:

```
JobState=PENDING Reason=Priority
```

This means the scheduler is prioritizing other jobs first.

Possible solutions:

- request fewer CPUs
- reduce memory
- shorten time limit

Example lighter request:

```
--cpus-per-task=2
--mem=10G
--time=00:30:00
```

---

# 15. Save Environment Snapshot

To preserve the working environment:

```bash
python -m pip freeze > $PROGAP_SCRATCH/env/pip_freeze_$(date +%Y%m%d).txt
```

---

# Recommended Daily Workflow

1. SSH into Bouchet  
2. Start `tmux`  
3. Request GPU node with `salloc`  
4. Activate the `progap` conda environment  
5. Load CUDA module  
6. Run experiments  
7. Submit large runs using `sbatch`

---
