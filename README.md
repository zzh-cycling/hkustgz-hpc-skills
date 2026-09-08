# HKUST-GZ HPC Skills

Agent skills for working on the HKUST-GZ High Performance Computing clusters. Each skill is a portable [`SKILL.md`](https://docs.claude.com/en/docs/claude-code/skills) (`name` + `description` frontmatter) and works in **both [Claude Code](https://docs.claude.com/en/docs/claude-code) and [Codex](https://github.com/openai/skills)**.

## Skills

| Skill | Description |
|---|---|
| [`hkustgz-hpc2`](./hkustgz-hpc2/) | Slurm cheat sheet for the HKUST-GZ HPC **Phase 2** cluster — job submission, partitions, GPU jobs, monitoring, cancellation, walltime sizing, and storage layout. |
| [`hkustgz-hpc4`](./hkustgz-hpc4/) | HKUST-GZ HPC **Phase 4** routing and operations — Kunpeng CPU Slurm jobs on `hpc` and Ascend 910C NPU containers in AIStudio. |
| [`hpc2ust-fibochain`](./hpc2ust-fibochain/) | Project-specific ops rules for FibonacciChain.jl Julia jobs on Phase 2 (`hpc2ust`) — Julia precompile stagger, architecture pollution, bad-node excludes, gap-fill data discipline, dual-repo sync. Personal companion to `hkustgz-hpc2`, not a general-purpose skill. |

Install only the phases your account can access. This keeps unavailable clusters out of the agent's choices. Users with both accounts can install both; the skills then route from the conversation, project, SSH target, and job files.

## Install

Each skill is a self-contained directory containing a `SKILL.md`. Choose the installation matching your account:

| Account access | Install |
|---|---|
| Phase 2 only | `hkustgz-hpc2` |
| Phase 4 only | `hkustgz-hpc4` |
| Both phases | both skills |

Prefer a project-local installation when a project always targets one phase.

### Claude Code

Copy the skill folder into a skills directory:

```bash
git clone https://github.com/isPANN/hkustgz-hpc-skills.git

# Personal — choose one, or run both commands if you have both accounts
cp -r hkustgz-hpc-skills/hkustgz-hpc2 ~/.claude/skills/
cp -r hkustgz-hpc-skills/hkustgz-hpc4 ~/.claude/skills/

# Or install the project's phase only
cp -r hkustgz-hpc-skills/hkustgz-hpc2 .claude/skills/
cp -r hkustgz-hpc-skills/hkustgz-hpc4 .claude/skills/
```

### Codex

Use the bundled `skill-installer` (installs into `$CODEX_HOME/skills`, default `~/.codex/skills`):

```bash
# Choose one, or run both commands if you have both accounts:
install-skill-from-github.py --repo isPANN/hkustgz-hpc-skills --path hkustgz-hpc2
install-skill-from-github.py --repo isPANN/hkustgz-hpc-skills --path hkustgz-hpc4
```

…or just clone and copy the folder yourself:

```bash
git clone https://github.com/isPANN/hkustgz-hpc-skills.git
# Choose one, or run both commands if you have both accounts:
cp -r hkustgz-hpc-skills/hkustgz-hpc2 ~/.codex/skills/
cp -r hkustgz-hpc-skills/hkustgz-hpc4 ~/.codex/skills/
```

Restart Codex to pick up new skills.

## License

[MIT](./LICENSE)
