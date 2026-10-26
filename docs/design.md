# NCHC-Bridge design

## Intended workflow

Edit locally, run a small smoke test, submit the current source and configuration to a remote GPU machine, monitor training, and pull selected results. The source can include uncommitted work; pushing to Git is optional. Record the Git revision and the actual transferred file hashes so an experiment can be traced to its source.

Assumptions to verify during setup: the local host can reach a remote SSH endpoint, remote storage is writable, and a permitted way to launch GPU jobs exists. For a cluster, establish whether a login/data-transfer node and compute nodes share storage. Do not infer an NCHC hostname, partition, project account, or GPU resource string from this repository's name.

## Architecture

| Component | Responsibility |
| --- | --- |
| Local CLI | Read project configuration, select files, validate paths, transfer, submit, and show results. |
| SSH transport | Authentication, jump hosts, connection reuse, remote commands. |
| rsync transport | Incremental file transfer, exclusions, progress, and interrupted-transfer recovery. |
| Remote run directory | Fixed source, manifest, launcher, logs, outputs, and run state. |
| Launcher adapter | Slurm on a managed cluster; tmux on a directly accessible GPU host. |
| Agent skill | Interpret a training request and operate the same workflow with existing tools or the future CLI. |

Use an ordinary local Python CLI with the standard library (`argparse`, `tomllib`, `subprocess`, `pathlib`, `json`) for the first implementation. Invoke the existing `ssh` and `rsync` binaries. No remote daemon, custom transfer protocol, database server, or open inbound local port is needed.

## Separate three kinds of files

| Files | Direction and policy |
| --- | --- |
| Source, configs, dependency locks | Local to remote; capture a fixed snapshot for each run. |
| Datasets, pretrained weights, model caches | Transfer/download separately; reuse named versions on remote storage. |
| Logs, metrics, checkpoints | Remote to local; select paths explicitly, with large checkpoints opt-in. |

Keep `.git`, virtual environments, caches, credentials, and training outputs out of source uploads. Include dependency declarations and locks. Do not copy a local CUDA/PyTorch environment to a different machine. Bootstrap a compatible environment on the remote host using the project's existing package manager or container workflow, accounting for the remote driver and site modules. A lockfile alone does not establish GPU compatibility.

Store non-secret host aliases and project settings in a project config; keep SSH keys and authentication in SSH configuration. Provide a local override for user-specific paths. A source selection policy should include Git-tracked files and explicitly selected untracked files, then apply bridge exclusions. Do not assume rsync exclusions interpret `.gitignore` syntax identically.

## Run isolation and repeatable transfer

Example remote layout:

```text
project-root/
  .incoming/<run-id>/source/      # incomplete upload; never executable as a run
  runs/<run-id>/
    source/                      # finalized source; later syncs never update it
    manifest.json
    launch.sh
    logs/
    outputs/
    state.json
  datasets/<dataset-version>/
  envs/<environment-id>/
  cache/
```

1. Generate a unique run ID and capture selected local files into a local staging directory. Verify files did not change during capture; fail and retry capture if they did. Exclude that staging directory from source selection.
2. Save source hashes, Git revision and dirty status, command arguments, dependency-lock hash, dataset versions, and target configuration in a manifest. Never serialize tokens or secret environment values.
3. Upload to `.incoming/<run-id>`. Resume an interrupted transfer using the same frozen local stage and run ID. A changed source snapshot gets a new ID.
4. After a successful transfer and manifest verification, rename the staging directory into `runs/<run-id>` on the same filesystem. Refuse to overwrite an existing finalized run. The rename is the publication boundary; rsync `--delay-updates` alone does not make a whole directory atomic.
5. Launch from that finalized source and write runtime output outside it. Read-only permissions can reinforce the convention when compatible with the training program. Keep writable runtime configurations in the run directory.
6. Record the job/session ID and lifecycle state locally and remotely. Later uploads get separate directories, even while earlier jobs run.

For the first implementation, uploading a fresh small source tree is acceptable. Then add rsync `--copy-dest` against a previous finalized source directory so unchanged files can be copied remotely and changed files can use a remote basis. Use an absolute remote basis path and validate behavior with deployed rsync versions. Avoid hard-linking writable source between runs. Hash verification costs remote reads, so measure it separately from transfer time.

## Proposed configuration and commands

The following schema and commands are design examples, not currently available interfaces:

```toml
schema_version = 1
project = "experiment"

[remote]
host = "gpu-remote"                # existing SSH alias
root = "/absolute/remote/project"  # shared with compute nodes if using Slurm
backend = "slurm"                  # or "tmux" on a direct GPU host

[environment]
setup_script = "scripts/remote-env.sh"

[sync]
exclude_file = ".bridgeignore"

[artifacts]
include = ["logs/**", "outputs/metrics.json"]

[slurm]
partition = "site-partition"
account = "site-project"
gres = "gpu:1"                     # replace with the site's GPU request
time = "04:00:00"
cpus_per_task = 8
memory = "64G"
```

```bash
bridge doctor                          # tools, access, storage, backend readiness
bridge sync --dry-run                  # inspect file selection and destination
bridge sync                            # create/finalize source snapshot
bridge run -- python train.py --config configs/train.yaml
bridge status RUN_ID
bridge logs RUN_ID --follow
bridge pull RUN_ID                     # configured small artifacts
bridge pull RUN_ID --include 'outputs/checkpoints/best.pt'
bridge cancel RUN_ID                   # explicit cancellation
```

`run` captures and syncs its own source unless given an explicit finalized snapshot ID. `sync` returns that ID. Serialize concurrent writes to one staging/run ID and reject traversal outside the configured roots. Use argument arrays locally and proper POSIX quoting for remote commands; SSH still passes a command string through a remote shell. Treat environment setup scripts as project code, with the same review expectations as the training command.

The first CLI should target Linux and macOS clients with compatible rsync, plus Linux remotes. Windows can use WSL initially. Detect required rsync options rather than assuming the OS supplies them.

## Job execution

For **Slurm**, sync through the site's permitted SSH/data-transfer endpoint into shared storage, generate a batch script, and submit with `sbatch --parsable`. Capture the returned job ID. Use `squeue` for queued/running jobs and `sacct` for terminal state and exit code where accounting is enabled. If accounting is unavailable, retain a job-written completion marker. A job disappearing from the queue does not prove success. Allocate the requested GPU type using verified site partition/GRES/constraint settings. Run training only inside the allocation. Slurm submits scripts but does not automatically stage their referenced source files. [Slurm sbatch documentation](https://slurm.schedmd.com/sbatch.html).

For **direct hosts**, run a wrapper in a detached tmux session. The wrapper records start time, exit code, and completion state in the run directory, and redirects training output to a log. A vanished tmux session does not prove training succeeded. Use a configured GPU selection; tmux provides persistence, not resource scheduling or exclusive GPU ownership.

Record `SUBMITTING` before a submission. If SSH disconnects after submission but before an ID is returned, mark the outcome `UNKNOWN`, reconcile remote state, and avoid blind resubmission. Remote run locks and a unique run identifier reduce duplicates, but cannot make a scheduler call and local state update one atomic transaction.

## Transfer performance

Start with rsync's incremental transfer over SSH and persistent remote datasets. rsync must be installed at both ends; it uses file metadata to skip unchanged files and can transfer differences against an existing destination/basis. [rsync documentation](https://rsync.samba.org/ftp/rsync/rsync.1).

- Reuse SSH connections with `ControlMaster auto`, a private `ControlPath`, and `ControlPersist`. Use existing `ProxyJump` settings where needed. This reduces repeated connection setup; it does not increase link bandwidth. [OpenSSH configuration](https://man.openbsd.org/ssh_config).
- Use a private relative `--partial-dir` to preserve interrupted transfers. Retry a bounded number of times for transient transport failures; stop for authentication errors, quota exhaustion, or incompatible tools.
- Enable rsync compression for compressible source when measurements justify it. Keep SSH compression off when rsync compresses. For compressed datasets and checkpoints, start with compression off. Negotiate newer compression features only after checking both rsync versions.
- Keep checkpoints, environments, and model caches outside source scanning. Pack large collections of tiny dataset files into reusable shards if metadata operations dominate.
- Benchmark before adding parallel streams. Parallel transfers can compete for disk and network bandwidth; only use them for independent dataset shards after measurement and site approval where required.

The ideal first-upload time is `bytes / effective throughput`; 10 GB at an effective 100 MB/s takes about 100 seconds before scanning, setup, and verification. Subsequent source uploads should usually involve much less data. Set latency targets after measuring the actual network, storage, and file count.

## Delivery milestones and acceptance checks

| Phase | Deliverable | Evidence it works |
| --- | --- | --- |
| 0: skill workflow | Establish access, preview selection, transfer, launch, inspect logs, retrieve metrics with existing tools. | A short authorized remote smoke run completes and produces retrievable results. |
| 1: CLI | Configuration, `doctor`, sync, one launcher, status, logs, pull, cancellation. | Source deletions appear correctly in new snapshots; edits after launch do not alter a running snapshot; results stay attached to their run. |
| 2: reliability | Transfer recovery, manifests, submission reconciliation, alternate launcher. | Interrupt an upload and resume it; simulate ambiguous submission without duplicating a job; propagate training failure and exit code. |
| 3: measured speed | Previous-snapshot reuse, selective artifacts, optional dataset sharding. | Compare first upload, one-file edit, no-change upload, and large-file resume with byte counts and elapsed time. |

Include failure checks for missing remote rsync, insufficient quota, paths containing spaces, symlinks outside the project, secret exclusions, a failed environment setup, unavailable GPU resources, a lost SSH session, and Slurm accounting being disabled. Use fake SSH/launcher processes for orchestration tests, then a real small transfer and short GPU smoke job on an authorized target. Avoid launching long training as a setup test.

Start with the launcher the actual machine uses. Add the second backend only when needed. Defer a GUI, continuous watch mode, bidirectional source merging, and automatic checkpoint mirroring until the basic submit/monitor/pull workflow is reliable.
