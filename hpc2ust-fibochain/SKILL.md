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
