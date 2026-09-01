---
name: hkustgz-hpc4
description: Operate the HKUST-GZ HPC Phase 4 domestic platform. Use when the request, project, or SSH target concerns `HPC4`/`四期`, `hpc4login.hpc.hkust-gz.edu.cn`, Kunpeng CPU, 910C/A3 NPU, or AIStudio. If this is the user's only installed HKUST-GZ HPC skill, use it for otherwise-unspecified HKUST-GZ cluster work without asking about another phase. Covers CPU Slurm jobs on partition `hpc` and NPU container development.
metadata:
  short-description: Slurm and AIStudio workflows on HKUST-GZ HPC Phase 4
---

# HKUST-GZ HPC4

Official documentation: https://docs.hpc.hkust-gz.edu.cn/docs/hpc4/domestic/

HPC Phase 4 has two distinct execution paths:

- **Kunpeng CPU work:** SSH to the login node and submit Slurm jobs to partition `hpc`.
- **Ascend 910C NPU work:** create a container development environment in the web portal's AIStudio. The official Phase 4 guide does not document NPU allocation through Slurm.

Do not reuse Phase 2 partition names, GPU directives, storage paths, or module assumptions.

## Phase selection

This skill's installation means Phase 4 is available to the user. Do not ask whether they have Phase 2 access. Keep using Phase 4 when it is the only installed HKUST-GZ HPC skill or is already established by the conversation, project, SSH target, job script, partition, or storage path.

When both phase skills are installed:

- Answer cluster-independent Slurm questions without choosing a phase.
- For phase-specific work, inspect available read-only context first: current hostname, SSH target/config, existing job script, partition, paths, and requested hardware. A new explicit target replaces older context.
- Ask which cluster the user uses only when a phase-specific action or command still cannot be chosen safely, or when the evidence conflicts. Once answered, retain it for the rest of the conversation.

Generic NPU/Ascend wording alone is not enough to distinguish Phase 2 from Phase 4; 910C/A3 or the Phase 4 AIStudio portal is.

## Confirm before consuming resources

Before `sbatch`, `srun`, or creating an AIStudio environment, show the complete resource request and command, then get explicit approval for that one action.

For Slurm, include:

- partition, walltime, nodes, tasks, CPUs per task, and memory
- job name, working directory, stdout, and stderr paths
- modules/environment and actual command

For AIStudio, include:

- image, resource specification, NPU chip count, Pod/node count, mounted model, and intended command
- both the portal's NPU quantity and the equivalent physical 910C card count

Do not carry approval to a later submission or environment creation.

## Connect

Browser portal:

https://hpc4login.hpc.hkust-gz.edu.cn/#/app/user

SSH login:

```bash
ssh <username>@hpc4login.hpc.hkust-gz.edu.cn
```

Use the login node for editing, inspecting resources, and submitting jobs. Run computation through Slurm or AIStudio.

## Kunpeng CPU: Slurm

The documented domestic CPU cluster has 22 Kunpeng nodes and one partition, `hpc`. Check live availability instead of assuming node state:

```bash
sinfo
module av
module list
```

Basic job script:

```bash
#!/bin/bash
#SBATCH --job-name=my_job
#SBATCH --output=%j.out
#SBATCH --error=%j.err
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=16G
#SBATCH --time=02:00:00
#SBATCH --partition=hpc

module load <software>/<version>
<command>
```

All `#SBATCH` directives must appear before the first non-comment command. Verify the requested module with `module av`; do not assume that Phase 2 modules exist on Phase 4.

Submit and manage jobs with standard Slurm commands:

```bash
sbatch script.sh
squeue -u "$USER"
scancel <jobid>
sinfo
```

Do not add Phase 2 GPU directives such as `--gres=gpu:N`: the Phase 4 Slurm guide documents this path for Kunpeng CPU jobs.

### Persistent interactive development

Keep the interactive allocation inside `tmux` so an SSH disconnect does not kill the shell:

```bash
tmux new -s debug_session
srun --job-name=interactive_debug \
  --nodes=1 \
  --ntasks=1 \
  --cpus-per-task=4 \
  --mem=16G \
  --time=02:00:00 \
  --partition=hpc \
  --pty bash
```

Detach with `Ctrl-b`, then `d`; reconnect with:

```bash
tmux attach -t debug_session
```

## Ascend 910C NPU: AIStudio containers

Use the browser portal, open **AIStudio** (add it from **应用中心** if absent), create a project, then create a development environment.

- Select a compatible container image.
- Select `NPU` and request an even number of NPU chips.
- One physical 910C card contains two 910B chips; a portal quantity of 16 means 8 physical 910C cards.
- Set the resource specification and Pod/node count. The default is one Pod; distributed training may require more.
- Wait for the environment to show `运行中`, then use its displayed SSH command, Jupyter, or VS Code entry.
- Verify allocation inside the container with `npu-smi info`.
- Each reported 910B chip has 64 GB memory.

Use the portal's current image/model list rather than copying a potentially stale name from this skill:

https://docs.hpc.hkust-gz.edu.cn/docs/hpc4/domestic/models-images/

For software or model compatibility, check the maintained lists:

- https://docs.hpc.hkust-gz.edu.cn/docs/hpc4/domestic/adaption-list-software/
- https://docs.hpc.hkust-gz.edu.cn/docs/hpc4/domestic/adaption-list-model/

## Do not invent undocumented cluster details

The Phase 4 guide does not specify storage mounts, quotas, billing tiers, maximum walltime, CPU cores per node, or Slurm NPU partitions. Inspect live state or ask the user instead of borrowing Phase 2 values.

## When NOT to use this skill

- HPC Phase 2 (`hpc2login`, A800/A40, `i64...` partitions, `/hpc2hdd`, `/hpc2ssd`) — use `hkustgz-hpc2`.
- HPC Phase 1 (`hpc1login`) — it uses LSF rather than this workflow.
