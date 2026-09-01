# Phase routing evals

Evaluate these cases without connecting to either cluster or consuming resources. Check semantic routing, not exact response wording.

| Installed skills | Context and request | Expected route | Ask phase? |
|---|---|---|---|
| `hkustgz-hpc2` only | “帮我写一个学校集群上的 Slurm 脚本” | Phase 2 | No |
| `hkustgz-hpc4` only | “帮我写一个学校集群上的 Slurm 脚本” | Phase 4 | No |
| Both | Existing script uses `i64m1tga800u`; monitor the job | Phase 2 | No |
| Both | Project SSH target is `hpc4login.hpc.hkust-gz.edu.cn`; request an interactive CPU allocation | Phase 4 | No |
| Both, no prior context | “`squeue` 怎么只看自己的任务？” | Generic Slurm answer | No |
| Both, no prior context | “帮我写一个学校集群上的 Slurm 脚本” | Target unresolved; do not choose cluster-specific parameters | Yes, one targeted clarification |
| Both, Phase 2 established earlier | “这个任务改用四期的 `hpc` 分区” | Phase 4; new explicit target overrides earlier context | No |
| Both, no prior context | “帮我申请一个 NPU 容器” | Generic NPU/Ascend is insufficient to select a phase | Yes, one targeted clarification |
| Both | “在 `hpc2login` 上申请 910C” | Conflicting evidence; do not act | Yes, one targeted clarification |
