# parallelcmd

A lightweight Python CLI for queueing and executing shell commands in parallel. Inspired by GNU Parallel, `parallelcmd` provides:

- **Command generation** from argument combinations
- **Concurrent execution** with live output and progress tracking
- **Job management**: inspect, reset, mark done, delete, update, and kill jobs
- **Flexible workflows**: resume and scale workers on demand, across nodes

It is a single Python file with no dependencies beyond the standard library.

## vs GNU Parallel

| | GNU Parallel | parallelcmd |
|---|---|---|
| **State** | Stateless¹ (fire and forget) | Stateful (SQLite queue persists) |
| **Resume** | Manual (`--joblog` + `--resume`) | Automatic (re-run `exec`) |
| **Job management** | Limited (joblog file) | First-class (`check`, `reset`, `delete`, `update`, `kill`) |
| **Dependency** | Perl | Python stdlib only |

Use GNU Parallel for one-shot parallel runs. Use parallelcmd when you need to stop, resume, inspect, and selectively retry jobs across sessions — especially for long ML or HPC experiment sweeps. See the [full comparison](comparison.md).

<small>¹ GNU Parallel can be made stateful via `--sqlmaster` / `--sqlworker`, but requires installing the Perl `DBD::SQLite` module separately.</small>

## Quick start

Create a job queue:

```bash
python3 parallelcmd.py init "echo {}" ::: a b c
```

Run queued jobs with 4 workers:

```bash
python3 parallelcmd.py exec -j 4
```

Or do both in one command:

```bash
python3 parallelcmd.py run -j 4 "echo {}" ::: a b c
```

If you omit the subcommand entirely, `parallelcmd.py` defaults to `run`:

```bash
python3 parallelcmd.py -j 4 "echo {}" ::: a b c
```

Check status:

```bash
python3 parallelcmd.py check
```

Next: [install it](install.md), then read [Concepts](concepts.md) to learn how commands are built and how job state is stored.
