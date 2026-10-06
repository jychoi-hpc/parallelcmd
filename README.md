# parallelcmd

A lightweight Python CLI for queueing and executing shell commands in parallel. Inspired by GNU Parallel, `parallelcmd` provides:

- **Command generation** from argument combinations
- **Concurrent execution** with live output and progress tracking
- **Job management**: inspect, reset, mark done, delete, update, and kill jobs
- **Flexible workflows**: resume and scale workers on demand, across nodes

📖 **Documentation: <https://jychoi-hpc.github.io/parallelcmd/>**

## vs GNU Parallel

| | GNU Parallel | parallelcmd |
|---|---|---|
| **State** | Stateless¹ (fire and forget) | Stateful (SQLite queue persists) |
| **Resume** | Manual (`--joblog` + `--resume`) | Automatic (re-run `exec`) |
| **Job management** | Limited (joblog file) | First-class (`check`, `reset`, `delete`, `update`, `kill`) |
| **Dependency** | Perl | Python stdlib only |

Use GNU Parallel for one-shot parallel runs. Use parallelcmd when you need to stop, resume, inspect, and selectively retry jobs across sessions — especially for long ML or HPC experiment sweeps.

> ¹ GNU Parallel can be made stateful via `--sqlmaster` / `--sqlworker`, but requires installing the Perl `DBD::SQLite` module separately.

## Installation

Download the single script and make it executable — no pip or dependencies required. Requires Python 3.8+.

```bash
curl -O https://raw.githubusercontent.com/jychoi-hpc/parallelcmd/main/parallelcmd.py
chmod +x parallelcmd.py
```

> Some systems ship an older `python3` (e.g. Python 3.6 on OLCF Frontier login nodes), where `exec` hangs silently. Check with `python3 --version`, and use e.g. `python3.11` if needed.

## Quick start

```bash
# create a queue of 3 jobs
python3 parallelcmd.py init "echo {}" ::: a b c

# run them with 4 workers
python3 parallelcmd.py exec -j 4

# or both in one step
python3 parallelcmd.py run -j 4 "echo {}" ::: a b c

# check status
python3 parallelcmd.py check
```

Common tasks:

```bash
python3 parallelcmd.py reset --nonzero        # requeue failed jobs
python3 parallelcmd.py reset --done --id 17   # mark a job done without running it
python3 parallelcmd.py kill --id 12           # stop a running job, on any node
python3 parallelcmd.py diagnose               # investigate a stuck queue
```

## Documentation

- [Concepts](https://jychoi-hpc.github.io/parallelcmd/concepts/): command model, placeholders, job states
- [Running jobs](https://jychoi-hpc.github.io/parallelcmd/usage/running/): `init`, `exec`, `run`, global options
- [Managing the queue](https://jychoi-hpc.github.io/parallelcmd/usage/managing/): `check`, `reset`, `delete`, `update`, `diagnose`
- [Stopping jobs](https://jychoi-hpc.github.io/parallelcmd/usage/kill/): `kill`, timeouts
- [Multiple workers and nodes](https://jychoi-hpc.github.io/parallelcmd/usage/multinode/)
- [Hooks](https://jychoi-hpc.github.io/parallelcmd/usage/hooks/)
- [Examples](https://jychoi-hpc.github.io/parallelcmd/examples/), [Troubleshooting](https://jychoi-hpc.github.io/parallelcmd/troubleshooting/), [FAQ](https://jychoi-hpc.github.io/parallelcmd/faq/)
- [Comparison with GNU Parallel](https://jychoi-hpc.github.io/parallelcmd/comparison/)

The documentation source is in [`docs/`](docs/). To preview it locally:

```bash
pip install -r docs/requirements.txt
mkdocs serve
```
