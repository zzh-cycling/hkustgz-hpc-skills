---
name: hkustgz-hpc2
description: Operate the HKUST-GZ HPC Phase 2 Slurm cluster. Use when the request, project, or SSH target concerns `HPC2`/`二期`, `hkustgz-hpc2`, `hpc2login.hpc.hkust-gz.edu.cn`, A800/A40, `/hpc2hdd`, or `i64...` partitions. If this is the user's only installed HKUST-GZ HPC skill, use it for otherwise-unspecified HKUST-GZ cluster work without asking about another phase. Covers job submission, monitoring, cancellation, and storage.
metadata:
  short-description: Slurm job submission & monitoring on the HKUST-GZ HPC Phase 2 cluster
---

# HKUST-GZ HPC2 (Slurm) Cheat Sheet

HPC Phase 2 uses **Slurm**. Phase 1 uses LSF (`jsub` / `#BSUB`), while Phase 4 has a different Slurm partition and a separate AIStudio NPU workflow. Do not mix them.

Official docs: https://docs.hpc.hkust-gz.edu.cn/docs/hpc12/

## Phase selection

This skill's installation means Phase 2 is available to the user. Do not ask whether they have Phase 4 access. Keep using Phase 2 when it is the only installed HKUST-GZ HPC skill or is already established by the conversation, project, SSH target, job script, partition, or storage path.

When both phase skills are installed:

- Answer cluster-independent Slurm questions without choosing a phase.
- For phase-specific work, inspect available read-only context first: current hostname, SSH target/config, existing job script, partition, paths, and requested hardware. A new explicit target replaces older context.
- Ask which cluster the user uses only when a phase-specific action or command still cannot be chosen safely, or when the evidence conflicts. Once answered, retain it for the rest of the conversation.

## Rule: confirm every parameter before submitting

When the user asks you to run something on this cluster, **list every `sbatch`/`srun` parameter and get explicit approval before invoking it.** No silent defaults — applies even on the free `debug` partition.

Always present, then wait:

- `-p` partition
- `-t` walltime
- `-n` / `-c` CPU count
- `--mem` memory
- `--gres=gpu:N` GPU count (+ partition implies type)
- The actual command(s) to run
- `module load …` / env activation
- `-o` / `-e` output paths
- `-D` working directory if it matters

Prior approval is scoped to that one submission — don't carry it forward to the next run. Use AskUserQuestion when there's a real choice to make (partition tier, walltime budget).

## Connect

```bash
ssh hkustgz-hpc2          # via ~/.ssh/config alias
# or
ssh <user>@hpc2login.hpc.hkust-gz.edu.cn
```

Home directory: `/hpc2hdd/home/<username>`

Login node is for editing/compiling/submitting only — never run heavy compute there. Use `srun` for interactive work.

### Load Slurm before any sbatch/srun/squeue

The login node's `/usr/bin/sbatch` etc. exist but are misconfigured (`slurm.conf` not at default path). You **must** load the Slurm module first or every Slurm command fails with `Unable to process configuration file`:

```bash
module load slurm
which sinfo   # → /opt/slurm/bin/sinfo  (correct one)
```

Make it permanent by appending to `~/.bashrc`:
```bash
echo 'module load slurm' >> ~/.bashrc
```

Compute nodes inside a job don't need this — they have a working local Slurm. The module is only for the login node where you submit/monitor from.

## Submit a batch job

Write a script, e.g. `run.sh`:

```bash
#!/bin/bash
#SBATCH -J myjob              # job name
#SBATCH -p i64m512u           # partition (see table below)
#SBATCH -o output_%j.txt      # stdout, %j = jobid
#SBATCH -e err_%j.txt         # stderr
#SBATCH -n 8                  # total CPU cores
#SBATCH --mem=16G             # memory
#SBATCH -t 02:00:00           # walltime HH:MM:SS

echo "Job started at $(date) on $(hostname)"
python your_script.py
echo "Job ended at $(date)"
```

Submit:
```bash
sbatch run.sh                 # → "Submitted batch job <jobid>"
```

### GPU job

```bash
#SBATCH -p i64m1tga800u       # A800 partition
#SBATCH --gres=gpu:1          # 1 GPU; use gpu:2 etc.
#SBATCH -n 8                  # CPU cores to pair with GPU
#SBATCH --mem=32G
#SBATCH -t 04:00:00
```

A800 nodes support up to 16 GPUs/node, A40 up to 8.

### Array job

```bash
#SBATCH -o output_%A_%a.txt   # %A=array id, %a=task id
#SBATCH --array=1-10          # or 1-10%3 to cap concurrency at 3
PARAM=$SLURM_ARRAY_TASK_ID
python your_script.py $PARAM
```

## Interactive session

```bash
srun -p i64m512u -n 4 --mem=8G -t 01:00:00 --pty bash
srun -p i64m1tga800u -n 8 --mem=32G --gres=gpu:1 -t 01:00:00 --pty bash
```

Land in a shell on a compute node. Exit with `exit` to release the allocation.

## Check status / monitor

```bash
squeue -u $USER                       # my running + pending jobs
squeue -j <jobid>                     # one job
scontrol show job <jobid>             # full details: paths, resources, why pending
sacct -u $USER --starttime=today      # today's history
sacct -j <jobid> --format=JobID,State,Elapsed,MaxRSS,ReqMem,AllocCPUS,AllocGRES
                                      # post-mortem: how much was actually used
tail -f output_<jobid>.txt            # live stdout from login node
ssh <nodename> top                    # `scontrol show job` shows NodeList; peek with top
```

State codes from `squeue`: `R`=running, `PD`=pending, `CG`=completing, `F`=failed, `CD`=completed.

If a job is stuck in `PD`, the `REASON` column tells you why (`Resources`, `Priority`, `QOSMaxJobsPerUserLimit`, etc.).

## Cancel / extend

```bash
scancel <jobid>                       # cancel one
scancel -u $USER                      # cancel all mine
scontrol suspend <jobid>              # pause (rarely needed)
scontrol resume <jobid>
```

Extending walltime is **not** a CLI op here — go to the platform portal → "My Jobs" → "Extend Time". Max +7 days per request, hard ceiling 14 days. For longer, file a ticket.

## Partitions (HPC2)

| Partition | Type | Cores/Node limit | Wall | Use for |
|---|---|---|---|---|
| `i64m512u` | CPU shared | up to 1024 | 7d | normal CPU jobs |
| `i64m512ue` | CPU exclusive | 128 | 7d | full-node CPU |
| `long_cpu` | CPU shared | 1024 | 14d | long runs |
| `emergency_cpu` | CPU high-pri | 512 | — | rush (premium) |
| `a128m512u` | AMD EPYC | 256 | 7d | AMD-optimized |
| `i64m1tga800u(e)` | GPU A800 | up to 16 GPU | 7d | training/big models |
| `i64m1tga40u(e)` | GPU A40 | up to 8 GPU | 7d | smaller GPU jobs |
| `i96m3tu(e)` | Large-mem | 192c / 3TB | 7d | memory-hungry |
| `debug` | Debug | 16c / 2 GPU | **0.5h** | **free** — for script validation |

`u` = shared (lower priority, lower rate), `ue` = exclusive node (higher).

### Priority = the partition you pick

You don't set priority with a flag. The partition's built-in tier decides both scheduling priority and price:

| Tier | Partitions | Use |
|---:|---|---|
| 100 (low) | `*u`, `long_*`, `i64m512r` | **default** — cheap, shared |
| 200 (med) | `*ue`, `i64m512re` | exclusive node, faster scheduling |
| 300 (high) | `emergency_cpu`, `emergency_gpu`, `emergency_gpua40` | rush, premium rate |

**HARD RULE: only ever propose low-tier (`*u`) partitions.** Medium (`*ue`) or high (`emergency_*`) are off-limits unless the user *explicitly* names that tier in their instruction for this specific job (e.g. "use `i64m512ue`" or "this is urgent, put it on emergency"). "Faster", "more reliable", "important" do not count as authorization — still propose `*u` and let the user override. This will rarely happen; assume `*u` by default.

**Don't add `--qos=…` or `--priority=…`** — this cluster doesn't use them (all QOS priorities are 0).

Live view on the cluster: `sinfo` (standard Slurm) — shows real-time node state per partition. (`jqueues` is HPC1/LSF only, don't use on HPC2.)

## Storage

Two GPFS pools:

| Mount | Type | Size | Use |
|---|---|---:|---|
| `/hpc2hdd` | HDD | 3.9 PB (often >90% full) | bulk, long-term, slower |
| `/hpc2ssd` | SSD | smaller | fast scratch I/O |

Home is `/hpc2hdd/home/<user>`. Inside it, several **symlinks** map portal-managed areas — pay attention to which point at SSD vs HDD:

| Symlink | Target | What it really is |
|---|---|---|
| `~/jhspoolers` | `/hpc2ssd/JH_DATA/spooler/<user>` | **SSD scratch** — the only fast-I/O area you get |
| `~/jhupload` | `/hpc2ssd/JH_DATA/spooler/<user>/.upload` | portal upload landing on SSD |
| `~/jhaidata` | `/hpc2hdd/JH_DATA/jhai_data/<user>` | AI data area (HDD), large datasets |
| `~/.jhshares` | `/hpc2hdd/JH_DATA/share/<user>` | shared with your group |
| `~/.filedownload` | `/hpc2hdd/JH_DATA/filedownload/<user>` | portal download staging |

### Where to put what

| Content | Path | Why |
|---|---|---|
| Code, configs, scripts, conda env | `~/` (home, HDD) | small, persistent |
| Read-only datasets (long-term) | `~/jhaidata` | big, HDD is fine |
| Training intermediates, checkpoints, unpacked dataset cache | **`~/jhspoolers` (SSD)** | hot-path I/O |
| Group-shared data | `~/.jhshares` | visible to your group |

### Using SSD scratch in a job

```bash
mkdir -p ~/jhspoolers/job-$SLURM_JOB_ID
cd ~/jhspoolers/job-$SLURM_JOB_ID
cp ~/jhaidata/big_dataset.tar.gz . && tar xf big_dataset.tar.gz
python ~/code/train.py --data ./extracted --ckpt-dir ./ckpts
# afterwards: archive final results back to HDD, then clean up SSD
cp ckpts/final.pt ~/results/
```

Rule of thumb: heavy random/repeat-read I/O on SSD, archival on HDD.

### Quotas

`quota` / `lfs quota` / `mmlsquota` are **not installed** on the login node — quota is only visible via the portal. Each pool (HDD/SSD) and each portal-managed area is counted separately. If a write fails with `Disk quota exceeded`, check the portal, not the filesystem.

## Baseline principle: validate before sbatch

Compute time is billed once the job starts. Burning a 4-hour GPU slot on a typo is avoidable. Before `sbatch`:

1. **Dry-run the script logic locally or in `srun --pty`** with tiny inputs.
2. **Smoke-test in `debug` partition first** — it's free and capped at 30 min. Submit the real script there with a small `--time` to confirm modules load, paths resolve, GPU is visible (`nvidia-smi`).
3. **Right-size walltime and resources.** Easier to extend (portal) or rerun than to overpay. After the first real run, use `sacct -j <jobid> --format=...MaxRSS,Elapsed` to calibrate future jobs.
4. **Sanity-check the submit script:** `bash -n run.sh` (syntax), confirm output/error dirs exist, confirm `--gres=gpu:N` only on GPU partitions.

This isn't a separate workflow — it's just the default before any non-trivial submission.

## Walltime: how to pick `-t`

It's an **upper bound**, not a fixed duration — script exits early, resources release immediately. Two forces pull opposite ways:

- **Too low** → `TIMEOUT` kills the job mid-run.
- **Too high** → lower scheduling priority, longer pending.

Billing is on actual elapsed time, not requested walltime, so overestimating doesn't cost more — just slower to start. Aim for **~1.5–2× your real estimate**.

First-time estimation:

1. **Time the smallest unit on `debug`** — 1 epoch / 1 iter / small slice:
   ```bash
   srun -p debug -n 8 --gres=gpu:1 -t 00:30:00 time python train.py --epochs 1
   ```
   Then `total ≈ unit_time × units × 1.5`.
2. **Calibrate from local**: A800 ≈ 1.5–2× a 4090; A40 ≈ 0.5–0.8× a 4090; HPC2 CPU ≈ 5–20× a laptop (per-core × core count).
3. **Always checkpoint** in long jobs (`torch.save` per epoch, etc.) — a TIMEOUT only costs the unsaved tail.
4. **After the first real run**, `sacct -j <jobid> --format=JobID,Elapsed,Timelimit` shows actual vs requested → tighten next time.

Format: `-t 30` (30 min) · `-t 02:00:00` (HH:MM:SS) · `-t 1-12:00:00` (D-HH:MM:SS). Use a dash, not space, between days and time.

## Common gotchas

- **Wrong partition for GPU**: `--gres=gpu:1` on a CPU partition → job fails or pends forever. Match partition to resource.
- **`-n` vs `-c`**: `-n` = number of tasks (MPI ranks), `-c`/`--cpus-per-task` = threads per task. For a single multi-threaded process use `-n 1 -c 8`, not `-n 8`.
- **Output file path not writable**: `-o`/`-e` paths are evaluated at submission time. Relative paths resolve against the submission directory unless `-D` overrides.
- **Modules not loaded inside the script**: login shell ≠ batch shell. `module load …` (or `source ~/.bashrc`) inside the script if you need it.
- **Walltime too short**: job gets `TIMEOUT` killed mid-run. Bump it, but not absurdly — pending priority depends on requested resources.
- **`squeue` shows nothing after sbatch**: the job already finished (or failed instantly). Check `sacct -j <jobid>` and the error file.

## When NOT to use this skill

- On HPC Phase 1 (`hpc1login`) — that's LSF, commands are `jsub` / `jjobs` / `jctrl` / `jhist` with `#BSUB` directives.
- On HPC Phase 4 (`hpc4login.hpc.hkust-gz.edu.cn`, Kunpeng CPU, 910C NPU, or AIStudio). Use `hkustgz-hpc4`.
- For the K8s / AI container workflow on HPC2 — that's a separate path (`/docs/hpc12/k8s/...`), not Slurm.
