---
name: nchc-bridge
description: Transfer local training code to remote GPU resources over SSH, launch jobs through Slurm or a direct host, inspect progress, and retrieve selected artifacts. Use for local-development and remote-training workflows, including NCHC targets.
---

# NCHC Bridge

Use existing SSH and rsync tools to move a training project from local development to remote GPU execution. A custom `bridge` CLI is optional; establish whether it actually exists before invoking it.

## Establish the target

Inspect the project README, dependency files, launch scripts, existing bridge settings, and relevant SSH aliases. Avoid printing private keys, tokens, or unrelated SSH configuration. Determine the source root, SSH endpoint, writable remote root, launch backend, environment setup, dataset locations, training command, and artifact selection from project context and the user's request. Ask only for missing details that block the requested action.

H100/H200 names do not identify the access method or scheduler configuration. Do not invent NCHC hostnames, partitions, project accounts, or GPU resource names. A scheduler login node is a submission/transfer endpoint; GPU training runs inside an allocation.

For a plan-only request, deliver the design without connecting or launching jobs. For an execution request, follow the user's authorized scope; preparing a bridge alone does not authorize a long training job. A configured target and an explicit training request normally suffice to proceed without repeated confirmation.

## Operate the bridge

Read [references/workflow.md](references/workflow.md) for preflight checks, transfer options, run isolation, launch adapters, and recovery. Apply the portion relevant to the request.

- Keep source, datasets, and outputs in separate locations. Reuse remote data and rebuild the environment from project dependency declarations; do not sync a local virtual environment.
- Preview source selection for first setup or a changed selection policy. Include intended uncommitted code and exclude credentials, environments, caches, datasets, and generated outputs. Inspect unusual symlinks instead of silently uploading external content or producing an incomplete source snapshot.
- Freeze selected local source before upload and stage it under a unique run ID. Finalize only after successful transfer and verification; later edits and syncs must not modify that run's source. Preserve a resumable stage on failure.
- Use the site's scheduler when present, or a persistent session on a directly accessible GPU machine. Capture a job/session ID and a training exit status; neither a successful SSH call nor an empty queue proves training succeeded.
- Retrieve requested logs, metrics, and checkpoints into a directory named for the run. Pull completed checkpoints or a snapshot made by the training process; do not copy actively changing checkpoint files as if they were consistent.

## Report the result

Distinguish prepared, transferred, submitted, running, and completed. Report the destination, source/run ID, job ID where available, launch command, observed state, and where logs/results can be found. State which checks actually ran and which still require target access. Avoid reporting success from a proposed command or a scheduler submission alone.
