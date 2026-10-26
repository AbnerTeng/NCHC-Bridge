# NCHC-Bridge

Develop and test on a local machine, then transfer code to remote H100/H200 resources and run training there.

Recommended foundation: **rsync over SSH, a fixed source snapshot for each run, and a remote job launcher**. Keep datasets and model caches on remote storage between runs. Rebuild the Python environment remotely from dependency files.

```text
Local project ── SSH + rsync ──► Remote source snapshot
                                      │
                                 Slurm or tmux
                                      │
                                 GPU training
                                      │
Local results ◄── selective pull ── Logs / metrics / checkpoints
```

This repository currently contains a design and a portable agent skill. The skill guides an agent through using existing SSH and rsync tools. The proposed `bridge` CLI is not implemented.

- [Design and implementation milestones](docs/design.md): architecture, command interface, transfer performance, and acceptance checks.
- [NCHC Bridge skill](skills/nchc-bridge/SKILL.md): instructions an agent can use to configure a project, sync it, launch training, and retrieve results with existing tools.
- [Operational workflow](skills/nchc-bridge/references/workflow.md): transfer options, snapshot handling, scheduler behavior, and recovery.

## Install from GitHub

Install the skill on your **local development machine**, where you use Codex and edit your training project. Choose either the Codex installer or the manual method below. You do not need to install the skill on the remote GPU machine.

Before starting, have Codex installed and access to this GitHub repository. The manual method also requires Git. To use the training workflow afterward, you will need SSH access to your GPU machine, rsync on both machines, and either Slurm access or tmux on a direct GPU host.

**Option 1: Install through Codex**

Paste this into the **Codex chat**, rather than your terminal:

```text
$skill-installer Install the nchc-bridge skill from GitHub repository
AbnerTeng/NCHC-Bridge, path skills/nchc-bridge, ref main.
```

The skill directory is [skills/nchc-bridge](https://github.com/AbnerTeng/NCHC-Bridge/tree/main/skills/nchc-bridge). Install the whole directory, including `references/` and `agents/`, rather than just `SKILL.md`. Installation does not require SSH credentials for your GPU machine and does not launch training.

Codex supports installing skills from other GitHub repositories through `$skill-installer`. If the newly installed skill does not appear, restart Codex. [Official OpenAI documentation](https://learn.chatgpt.com/docs/build-skills).

**Option 2: Install manually**

Run these commands in a terminal on Linux, macOS, or WSL, from the directory where you want to keep the repository checkout:

```bash
git clone https://github.com/AbnerTeng/NCHC-Bridge.git
mkdir -p "$HOME/.agents/skills"
cp -R NCHC-Bridge/skills/nchc-bridge "$HOME/.agents/skills/"
```

These are first-install instructions. If you already have a local copy of the skill, review its changes before replacing it. For a repeatable version, ask the installer to use a published tag or commit instead of `main`.

Codex discovers user skills in `~/.agents/skills`. Installing the skill there makes it available while working in other training repositories. [Skill discovery documentation](https://learn.chatgpt.com/docs/build-skills).

**Verify installation**

In Codex CLI or the IDE extension, run `/skills` and look for `nchc-bridge`, or type `$` and select the skill. If it does not appear, restart Codex and check again. Then open your training project and use the example below.

## Use it in your training project

Open your training project in Codex on the local machine that can reach your GPU server. You need SSH access to that server, rsync on both machines, and either Slurm access or tmux on a directly accessible GPU host. Each user supplies their own target and authentication; this repository does not provide GPU access.

Start with a setup and transfer preview, replacing the example values:

```text
$nchc-bridge Prepare this project for remote training.

Local project: /path/to/my-training-project
Remote SSH alias: gpu-remote
Remote project directory: /remote/path/my-training-project
Launch method: direct SSH with tmux
Environment setup: source /remote/path/venv/bin/activate
Dataset already on remote: /remote/path/datasets/my-dataset
Training command: python train.py --config configs/train.yaml

Check access and dependencies, then preview the source transfer.
```

For a scheduled cluster, use `Launch method: Slurm` and supply the partition, account, GPU request, and environment/module setup if known. H100/H200 hardware alone does not identify those settings. If an environment has not been created yet, say so and let the agent inspect the project's dependency files.

Once configured, request the actual work:

```text
$nchc-bridge Transfer the current code and launch the training command.
Keep datasets on the remote machine. Show the run ID, job/session ID,
and log location.
```

Follow up with requests such as:

```text
$nchc-bridge Check the latest run and show the last 100 log lines.
```

```text
$nchc-bridge Download the metrics and best completed checkpoint from that run.
```

If you only want to try the workflow from a checkout, ask the agent to read `skills/nchc-bridge/SKILL.md` and follow it. No installation is needed for that explicit file-based request.

## Sharing this repository

Keep the complete `skills/nchc-bridge/` directory committed and pushed alongside this README. The GitHub installer can only download files present in the selected remote branch, tag, or commit. Users need access to the repository; a private repository requires their own GitHub authentication.

Share the installation prompt above with Codex users. For broader distribution through the ChatGPT/Codex plugin directory, this skill can later be packaged as a plugin. [Distribution guidance](https://learn.chatgpt.com/docs/build-skills).
