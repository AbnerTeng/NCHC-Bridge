# Remote training workflow

## Preflight and project selection

Establish an SSH alias and test access using the user's existing authentication. Preserve host-key verification and existing jump-host settings. Interactive MFA may require the user to establish a session; repeated automated authentication attempts will not resolve it. Do not install keys or rewrite global SSH configuration merely to prepare a project.

Check local and remote rsync versions, remote shell behavior, writable storage, space/quota, and the selected launcher. For a scheduled cluster, establish the shared filesystem path, scheduler account/partition/GPU request, and allowed environment modules. Probe GPUs from a permitted allocation rather than expecting `nvidia-smi` on a login node. For a direct GPU host, inspect device visibility and existing utilization without claiming a free device is reserved.

Use the repository's training command and package manager. Read dependency locks and any remote setup script. Confirm the remote Python/PyTorch/CUDA stack during a short smoke job if the user requested one. Rebuild environments in a remote environment directory or use an existing compatible site container. Environment creation and dataset downloads can be expensive; account for those costs separately from source transfer.

For source selection, preserve tracked files and the user's intended untracked code/configs. Apply explicit bridge exclusions after Git-based selection: tracked files can still contain unwanted credentials or generated output. Typical exclusions include `.git/`, `.venv/`, `venv/`, `__pycache__/`, `node_modules/`, `.env`, `.env.*`, `.aws/`, `.ssh/`, private keys, dataset directories, model caches, checkpoints, and experiment outputs. Include a sanitized `.env.example` only when useful. Adapt directory names to the project rather than blindly excluding code named `data` or `models`. Git ignore files and rsync filter files have different semantics.

## Capture and upload source

Generate a unique run ID. Capture the selected files and a manifest in a local stage outside the selected source root. Capture must detect concurrent edits and fail rather than silently combine changing file versions. Record the final stage's relative paths, file hashes, permissions and symlink targets as needed, revision/dirty status, environment lock hash, dataset identifiers, and launch arguments. Exclude secrets from manifests. Keep the stage until the transfer finishes or is explicitly abandoned.

Create an incomplete remote directory `.incoming/<run-id>/source/` with private permissions under the configured project root. Validate the destination is within that root. Treat paths as data: use local subprocess argument arrays and correctly quote remote shell strings. For a handcrafted shell recipe, restrict identifiers to simple names and use explicit absolute paths; generate a quoted script for more complex paths.

The following is a command template, to be adapted after source selection and SSH setup. `LOCAL_STAGE` must already contain only the chosen source files, `REMOTE_STAGE` is the absolute incomplete `source` directory, and `SSH_ALIAS` identifies the target. The destination example assumes a remote path without whitespace or shell metacharacters; use compatible protected arguments and correct quoting for other paths.

```bash
rsync -rlptn --itemize-changes --partial-dir=.rsync-partial \
  -e ssh "$LOCAL_STAGE/" "$SSH_ALIAS:$REMOTE_STAGE/"

rsync -rlpt --itemize-changes --stats --partial-dir=.rsync-partial \
  -e ssh "$LOCAL_STAGE/" "$SSH_ALIAS:$REMOTE_STAGE/"
```

Both ends require rsync. Choose version-compatible progress and compression options when useful. Recursive file/permission/time transfer is sufficient for source; avoid requiring owner/group preservation on shared systems. Inspect external/absolute symlinks before capture and choose whether to reject or intentionally include their content. A symlink into another machine's filesystem can break training even if rsync faithfully copies it.

No remote deletion is needed for a fresh stage. Resume only with the same frozen local stage; never reuse an ID for different source. When a previous finalized source is available, consider an absolute remote `--copy-dest` basis to reuse unchanged files without sharing writable hard links. Verify support and behavior before enabling the optimization.

Use compression for source only when worthwhile; it often adds little for compressed weights or archives. Keep SSH compression off if rsync handles compression. Retain the partial directory on transfer failure. Exit code 24 or another nonzero result from a frozen stage is a failure to investigate, not a successful snapshot.

After upload, verify the manifest against the remote source. Put the manifest and generated launcher alongside `source/` in the incomplete run directory. Publish by renaming `.incoming/<run-id>` to `runs/<run-id>` on the same filesystem while holding a per-run lock, refusing an existing destination. Do not confuse per-file delayed updates with an atomic source snapshot. Keep partial-transfer directories out of the finalized source after success.

Later submissions always use a new finalized directory. Output belongs in `logs/` and `outputs/` outside `source/`; configure the training program accordingly. Use a working copy within the run if the training framework requires writing beside its source. Do not edit shared dependencies or datasets while jobs rely on them; use environment and dataset versions.

## Slurm adapter

Generate a batch script using verified site settings. It should request the intended GPU resources, CPU/memory/time limits, set a known working directory, run environment setup with failure propagation, and then execute the training command. Put `#SBATCH` directives before executable shell statements. Shell variables in directives are not expanded; use rendered literal values or submission arguments. Ensure the log directory exists before submitting, because Slurm opens logs before the script runs.

Use `sbatch --parsable` and store the numeric job ID; retain a cluster suffix separately if returned. A unique run ID in the job name/comment can aid reconciliation but is not itself a uniqueness lock. Use a per-run submission lock and persist submission intent. If the call disconnects after possible acceptance, inspect recorded state and scheduler jobs before retrying. Treat unresolved outcomes as unknown, not failed submissions ready for automatic repetition.

Use `squeue` to observe pending/running jobs. Use `sacct` to inspect final state and exit code when available. If accounting is disabled, arrange a job-written completion marker with the training exit code, and report unknown when neither source can confirm completion. A timeout, cancellation, or node failure may prevent any shell marker. Submit only the requested number of jobs; do not automatically requeue failed training unless that policy was requested.

For multi-GPU training, preserve the project's intended `torchrun`/distributed launcher and coordinate it with Slurm tasks. Do not assume increasing a GPU count automatically parallelizes a plain Python script. Add multi-node launch support only when needed by the project.

## Direct-host adapter

Use a detached tmux session with a unique run name and a wrapper script that changes to the finalized source, activates the environment, redirects output into the run logs, and records the actual command exit status. A wrapper should catch environment-setup failure too. Record the session name and timestamps before returning to the user.

The session survives an SSH disconnect. It does not allocate a GPU or survive every host failure. Respect the host's resource policy; set a requested `CUDA_VISIBLE_DEVICES` value only after checking the user's intended GPU selection. If tmux is unavailable, determine whether a site-supported persistent launcher exists before installing anything or substituting another mechanism.

Use logs and explicit completion state to determine success. On cancellation, target the recorded run session/process group so unrelated GPU jobs remain unaffected.

## Artifacts and failure recovery

Pull logs and metrics by default and checkpoints when requested. Preserve the run ID in the local destination and never overwrite local source with remote artifacts. Capture a consistent checkpoint by waiting for an atomic checkpoint publication or making a training-supported snapshot. Keep datasets/model caches remote unless their transfer is explicitly requested.

Retry interrupted transfers against the original frozen stage. Use a small bounded retry count for transient connection failures and stop for credentials, permissions, quota, or validation errors. Preserve run/job IDs across observation retries. Reconcile an uncertain job submission before any resubmission.

Useful verification cases are a first transfer, a no-change transfer using a remote basis, a single edit, a local deletion in the next snapshot, interruption/resume, failed training, and local editing during a remote run. Measure setup, file scanning, transferred bytes, verification, scheduler wait, and training time separately.

## Primary references

- [rsync manual](https://rsync.samba.org/ftp/rsync/rsync.1): transfer options, exclusions, remote basis directories, and partial files.
- [OpenSSH configuration](https://man.openbsd.org/ssh_config): host aliases, `ProxyJump`, `ControlMaster`, and `ControlPersist`.
- [Slurm sbatch](https://slurm.schedmd.com/sbatch.html): submission, script directives, output paths, and job IDs.
