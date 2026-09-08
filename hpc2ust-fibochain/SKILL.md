---
name: hpc2ust-fibochain
description: >-
  Submit, monitor, and collect FibonacciChain.jl Julia jobs on the HKUST-GZ HPC2
  Slurm cluster (hpc2ust). Use whenever the request involves `hpc2ust`,
  `hpc2login.hpc.hkust-gz.edu.cn`, `~/FibonacciChain.jl`, partitions
  `i64m512u`/`i64m512r`/`i64m512ue`/`long_cpu`/`a128m512u`/`i96m3tu`, sbatch for
  Julia/MPS/DMRG/TCI/Potts/lyapunov/topological-charge jobs, or collecting/syncing
  their data to the local NoisyFibonacciChain / MonitoredFibochainPaper repos.
  Complements the generic hkustgz-hpc2 skill with project-specific hard rules
  learned from past failures.
metadata:
  short-description: Project-specific ops rules for FibonacciChain.jl jobs on hpc2ust
---

# hpc2ust × FibonacciChain.jl Operations

> Canonical source: `hpc2ust-fibochain/` in
> [zzh-cycling/hkustgz-hpc-skills](https://github.com/zzh-cycling/hkustgz-hpc-skills).
> Edit there and re-copy/symlink to installed locations; do not patch local copies.

Cluster basics (partitions, storage, walltime, sacct) are covered by the generic
`hkustgz-hpc2` skill. This file holds **project-specific rules learned from real
failures** — they override generic advice where they conflict.

## Fixed facts

- SSH alias: `ssh hpc2ust`. Slurm commands must run via `bash -lc` (module env),
  e.g. `ssh hpc2ust 'bash -lc "squeue -u zzhi359"'`.
- Remote repo: `/hpc2hdd/home/zzhi359/FibonacciChain.jl`. Logs live in the repo
  root as `job-<name>-<jobid>.txt`.
- Local repos that receive data (see "Data sync" below):
  - `/Users/cycling/Documents/projects/NoisyFibonacciChain`
  - `/Users/cycling/Documents/projects/MonitoredFibochainPaper` (mirror — **every**
    download must also be copied there)

## Julia precompile discipline (top source of mass job death)

Multiple Julia jobs starting simultaneously race on the precompile pidfile and die
within ~1 minute with precompile errors. Rules:

1. **Stagger submissions by 45–50 s** whenever submitting more than one Julia job.
2. Every sbatch script must warm the cache before computing:
   ```bash
   julia --project=. -e 'using Pkg; Pkg.precompile()'
   julia --project=. -p $WORKERS exm/.../driver.jl ...
   ```
3. **Architecture pollution**: `a128m512u` is AMD znver3; cpu1/cpu2 nodes are Intel.
   They share the user's precompile cache. Submit a128 jobs **last**; wait until
   they finish AND nothing else is queued, then run a cache-rebuild
   (`Pkg.precompile()`) job before submitting to Intel partitions again.
4. Symptom check: if new jobs die in <1 min, read the log — precompile/pidfile
   errors mean re-stagger and resubmit.

## Parallelism patterns: shell-level vs ClusterManagers

Within one sbatch allocation there are two ways to fan work out. Both still obey
the precompile rules above — the pidfile race exists between **processes**, not
just between jobs, and `$JULIA_DEPOT_PATH` lives on shared GPFS, so processes on
*different nodes* race too.

### A. Shell-level: independent Julia processes (our default)

Complete multi-node sbatch script — one Julia process per node, each saturated
with local `-p` workers. Follows the `hkustgz-hpc2` `-n`/`-c` rule: `-n` counts
**processes**, `-c` is CPUs **per process**:

```bash
#!/bin/bash
#SBATCH -J cft_fill
#SBATCH --nodes=3                  # 3 independent Julia processes
#SBATCH --ntasks-per-node=1        # ONE process per node (not 64!)
#SBATCH --cpus-per-task=64         # each process gets 64 CPUs → -p 64 workers
#SBATCH -o job-cftfill-%j.txt
#SBATCH --exclude=cpu1-1,cpu1-35,cpu1-48,cpu1-81,cpu1-92,cpu1-95,cpu2-17

cd /hpc2hdd/home/zzhi359/FibonacciChain.jl
export JULIA_WORKER_TIMEOUT=300

# Warm the cache ONCE before launching anything — all nodes share
# $JULIA_DEPOT_PATH on GPFS, so concurrent cold starts race the pidfile.
julia --project=. -e 'using Pkg; Pkg.precompile()'

LO=(1 1001 2001); HI=(1000 2000 3000)   # per-node seed ranges
i=0
for node in $(scontrol show hostnames "$SLURM_JOB_NODELIST"); do
  srun --nodes=1 --ntasks=1 --nodelist="$node" \
    julia --project=. -p "$SLURM_CPUS_PER_TASK" \
      exm/Bulk_measure/driver.jl "${LO[$i]}" "${HI[$i]}" \
    > "log_${SLURM_JOB_ID}_chunk$((i+1)).txt" 2>&1 &
  i=$((i+1))
  sleep 20   # stagger cold-start processes: they race the pidfile too
done
wait
```

- `srun` here launches **job steps inside the existing allocation**, one pinned
  per node via `--nodelist`; each step inherits `--cpus-per-task=64`, and
  `$SLURM_CPUS_PER_TASK` hands the same number to Julia's `-p`. `-c` and workers
  always match — more CPUs than workers does NOT speed anything up.
- Do NOT set `--ntasks=192` / `--ntasks-per-node=64`: `-n` is process count
  (MPI-rank semantics per `hkustgz-hpc2`); Julia parallelism comes from `-p`
  inside each process, not from task count.
- `%j` only expands in `#SBATCH` directives — inside the script use
  `$SLURM_JOB_ID` for per-step log names.
- Multi-node allocation caveat: the job starts only when ALL nodes are free, and
  one bad node in the nodelist hurts every chunk sharing the job — the exclude
  list is mandatory. When chunks are independent, N staggered single-node jobs
  (this project's usual mode) schedule faster and fail smaller than one
  N-node job; use the multi-node form only when you specifically want one jobid.
- Work partitioning is manual (seed ranges / seedlists); each process owns a range.
- **Fault isolation**: one process crashes → the rest finish; resubmit only the
  missing range. This is what makes gap-filling cheap.
- No cross-process communication → no serialization pitfalls, no master to die.
- Tail waste: fixed ranges mean the slowest chunk sets the finish time; mitigate
  with smaller chunks, not with fancier parallelism.

### B. Julia-level: ClusterManagers SlurmManager (one master, workers as Slurm tasks)

```julia
using ClusterManagers, Distributed
const PROJECT_DIR = dirname(Base.active_project())
const NWORKERS = parse(Int, get(ENV, "SLURM_NTASKS", "64"))
# SLURM_CPUS_PER_TASK gives threads per task; we run --threads=1
addprocs(SlurmManager(NWORKERS); exeflags="--project=$PROJECT_DIR --threads=1")
pmap(work; batch_size=1)   # dynamic load balancing
```

Matching sbatch shape — the mirror image of A: here `-n` IS the worker count,
each task is single-CPU (`--threads=1`), and `SlurmManager` launches the workers
as its own srun steps inside the allocation:

```bash
#SBATCH --nodes=8
#SBATCH --ntasks-per-node=8        # 64 worker tasks total (master runs on the batch shell)
#SBATCH --cpus-per-task=1          # one CPU per Julia worker; NOT 64
```

- One sbatch job spans many nodes; pmap with `batch_size=1` balances dynamically —
  good when per-task cost varies wildly.
- Costs and failure modes (all observed or one misstep away):
  - Master is a **single point of failure**: master lands on a bad node → the whole
    multi-node allocation dies at once. Pattern A loses one chunk; B loses everything.
  - A worker dying mid-pmap can hang the whole job.
  - Every function workers touch must be `@everywhere`-defined. A master-local
    function → `UndefVarError: #f not defined`, all workers exit in ~40 s having
    computed nothing (real incident, 15 jobs burned).
  - Needs `JULIA_WORKER_TIMEOUT=300` — workers cold-start slowly on GPFS.
  - `addprocs` launches all workers at once → same pidfile race; the sbatch-level
    warm-up must complete before the master starts.
  - `ClusterManagers` must be in the Project env; don't add it ad hoc.

### Which one, when

- **Default to A.** Our workload is sample generation at fixed (L, χ, γ) → per-seed
  cost is near-uniform → dynamic balancing buys nothing, and A's fault isolation +
  trivial restartability win. Multiple independent single-node jobs with stagger
  IS pattern A across the queue.
- Use B only when per-task cost varies by orders of magnitude AND chunks can't be
  pre-split sensibly, or the algorithm is genuinely distributed. Do not reach for B
  to fix imbalance that smaller chunks would fix more robustly.
- Never mix the two casually: B inside a `for` loop of processes multiplies the
  precompile race and the debugging surface for no benefit.

## Partition playbook (project usage)

| Partition | Use for | Notes |
| --- | --- | --- |
| `i64m512u` | workhorse CPU jobs | often fully occupied; check `sinfo` first |
| `i64m512r` | same, spillover | check idle nodes before relying on it |
| `i64m512ue` | **collect/merge only** | expensive; never for bulk compute unless user says so |
| `long_cpu` | long runs, spillover | |
| `a128m512u` | last resort | architecture pollution, see above |
| `i96m3tu` | big-memory jobs (Y-matrix, projected ytau) | few idle nodes; may need patience |
| `debug` | free 0.5 h validation, collect/merge when queues are full | |

- Bad nodes — always add:
  `--exclude=cpu1-1,cpu1-35,cpu1-48,cpu1-81,cpu1-92,cpu1-95,cpu2-17`
- **Match `-c` to Julia workers** (`-c 64` → `-p 64`). More CPUs than workers does
  NOT speed things up.
- `QOSMaxJobsPerUserLimit` on pending jobs → scancel and resubmit to a partition
  with headroom; don't leave them sitting.

## Remote scripting discipline

- **Never use nested/inline heredocs over ssh** to create remote scripts. `\$VAR`
  and `$VAR` get expanded by the wrong shell and you get empty variables (happened
  twice). Write the script locally, `scp` it up, run it.
- For batch submissions with stagger, run the loop **on the login node under
  nohup**, not from the local ssh session — a dropped ssh then cannot orphan or
  kill the loop:
  ```bash
  ssh hpc2ust 'bash -lc "cd ~/FibonacciChain.jl && nohup bash submit_loop.sh > submit_loop.log 2>&1 &"'
  ```
- To STOP a batch-submission loop: kill the remote process
  (`ps -u zzhi359 | grep sub`), not just the local ssh — otherwise it keeps
  submitting as a zombie.
- pmap drivers: any function referenced by workers must be `@everywhere`-defined.
  A master-local function gives `UndefVarError: #f not defined` and every job
  exits in ~40 s having computed nothing.

## Data rules (user's iron rules)

1. **Never recompute existing data.** Drivers with `isfile → skip` are safe for
   direct resubmission; drivers WITHOUT skip (e.g. cft sample generation) must go
   through a seedlist gap-fill driver that computes only missing seeds.
2. rsync/scp from remote: use **absolute paths**
   (`/hpc2hdd/home/zzhi359/...`) — `~` does not expand inside rsync's quoted
   remote path.
3. **Every downloaded file is also copied to MonitoredFibochainPaper**, preserving
   relative path under `data/`. Local conventions:
   - tci/potts ensembles: flat in `data/Bulk_measure/monitored_dynamics_{tci,potts}/gamma0.95/`
   - topological charge sharpening: `data/Bulk_measure/topological_charge_sharpening/Born/L{L}/gammaind10/t{T}/`
   - lyapunov: `data/Bulk_measure/lyapunov_spectrum_{y1,ytau}/L{L}/` (ensemble lives
     one level ABOVE the `chi*` raw-data dir)
   - Lyapunov data goes ONLY to MonitoredFibochainPaper.
4. Trajectory-file schema migrations: use `upgrade_trajectory_keys.jl` (remote:
   `~/upgrade_old_trajectory_files.jl`, local copy in `tmp/`) to add missing keys
   as `NaN` in `r+` mode — never fabricate values, never recompute.

## Collect commands (templates)

```bash
# cft (tci/potts) mps
julia --project=. exm/Bulk_measure/monitored_dynamics_cft.jl {tci|potts} collect_mps L 10 CHI [periods]
# lyapunov sector
julia --project=. exm/Bulk_measure/lyapunov_spectrum_sector.jl {y1|ytau} collect mps L 10 CHI
```

Prefer `debug` or `i64m512ue` nodes for collect/merge when compute partitions are
full. A job's success marker is the `done: ok=N skip=M failed=0` line in its log —
always grep it before declaring completion.

## Monitoring cadence

- After any batch submission, verify with `squeue` that jobs reached RUNNING and
  survived >5 min (precompile deaths happen in the first minute).
- For long batches, schedule a progress check (cron) rather than blocking.
- Report to the user with concrete counts: `ls <dir> | wc -l` vs target.
