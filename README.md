# parallelcmd

A lightweight Python CLI for queueing and executing shell commands in parallel. Inspired by GNU Parallel, `parallelcmd` provides:

- **Command generation** from argument combinations
- **Concurrent execution** with live output and progress tracking
- **Job management**: inspect, reset, mark done, delete, update, and kill jobs
- **Flexible workflows**: resume and scale workers on demand

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

Download the single script and make it executable — no pip or dependencies required.

```bash
# wget
wget https://raw.githubusercontent.com/AI-ModCon/parallelcmd/main/parallelcmd.py
chmod +x parallelcmd.py

# curl
curl -O https://raw.githubusercontent.com/AI-ModCon/parallelcmd/main/parallelcmd.py
chmod +x parallelcmd.py
```

You can then run it directly:

```bash
./parallelcmd.py --help
```

Or place it somewhere on your `PATH` (e.g. `~/.local/bin/`) to use it as `parallelcmd.py` from any directory.

## Requirements

- Python 3.8+
- Standard library only (no external Python dependencies)

Some systems ship an older `python3` (e.g. Python 3.6 on OLCF Frontier login nodes). Check with `python3 --version`, and use a newer interpreter if needed, e.g. `python3.11 parallelcmd.py ...` or `module load cray-python`.

## Quick start

```bash
python3 parallelcmd.py --help
```

Create a job database:

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

## Command model

`init` builds commands and stores them in `pardb.sqlite` by default. Use `--db <name>` to target `<name>.sqlite`, or set the `PARDB` environment variable. `init` (and `run`) refuse to touch an existing queue unless you pass `-a/--append` (add jobs) or `-f/--force` (drop and recreate).

- `:::` starts an inline argument list.
- `::::` starts an argument list loaded from a file (one value per line; empty lines and `#` comments are ignored).
- `:::: -` reads the argument list from stdin.
- Multiple lists are combined with Cartesian product.
- If the command has no argument placeholders, one `{}` per argument list is appended automatically (`init "echo" ::: a b ::: x` gives `echo a x`, `echo b x`). `{%}` and `{#}` don't count, so `"echo {%}" ::: a b` gives `echo {%} a`, `echo {%} b`.
- If no `:::` or `::::` separator is given and stdin is a pipe, stdin lines are used as the argument list automatically. If the command uses `{0}`, `{1}`, …, each line is split on whitespace into that many fields and the fields are paired as with `--zip` (the last field keeps any remaining text).

### Placeholders

| Placeholder | Meaning |
|---|---|
| `{}` | Current argument (positional, auto-assigned left to right) |
| `{0}`, `{1}`, … | Explicit positional argument from the Nth `:::` / `::::` list (0-indexed) |
| `{%}` | Worker slot number (0-indexed, stable for the lifetime of `exec`) |
| `{#}` | Job sequence number (the DB `Seq` of the job being run) |

`{}` and `{0}` / `{1}` can be mixed freely. `{}` takes the next list each time it appears, so a command using the same argument twice needs `{0}` (e.g. `"cp {0} {0}.bak"`). `{%}` and `{#}` are substituted at run time, not at `init` time, so the stored command retains the literal placeholder.

Commands are expanded with Python `str.format`, so **literal braces must be doubled**: write `awk '{{print $1}}'` to get `awk '{print $1}'`. A single `{print $1}` fails with `KeyError`.

Example — Cartesian product with implicit placeholders:

```bash
python3 parallelcmd.py init "python train.py --lr {} --seed {}" ::: 1e-3 1e-4 ::: 1 2 3
```

This creates 6 jobs.

Example — explicit positional placeholders (same result, order made explicit):

```bash
python3 parallelcmd.py init "python train.py --lr {0} --seed {1}" ::: 1e-3 1e-4 ::: 1 2 3
```

Example — GPU assignment via worker slot:

```bash
python3 parallelcmd.py run -j 4 "CUDA_VISIBLE_DEVICES={%} python train.py --lr {}" ::: 1e-3 1e-4 1e-5 1e-6
```

Each worker holds a fixed slot (0–3), so all its jobs run on the same GPU.

Example — `--zip` to pair lists element-by-element instead of Cartesian product:

```bash
# Without --zip: 4 jobs (a+x, a+y, b+x, b+y)
python3 parallelcmd.py init "cmd {0} {1}" ::: a b ::: x y

# With --zip: 2 jobs (a+x, b+y)
python3 parallelcmd.py init --zip "cmd {0} {1}" ::: a b ::: x y
```

Stops at the shortest list when lengths differ.

## Job states

Each job is a row in the `parjob` table. Its state is stored in `Exitval`, which is what `--where` filters match on:

| State | `Exitval` |
|---|---|
| Pending | `NULL` |
| Running | `-1000` |
| Running, `kill` requested | `-1001` … `-1164` (see [`kill`](#kill)) |
| Success | `0` |
| Failed | the command's exit code (`> 0`) |
| Timed out (`--timeout`) | `124` |
| Killed by `kill` | `-signum`, e.g. `-15` for `TERM` |

Other columns: `Seq` (job ID, used by `--id` and `{#}`), `Command`, `Starttime`, `Hostname` and `PID` (of the last run), `JobRuntime` (seconds).

## Subcommands

### `init`

Initialize the job queue, or append to an existing one.

```bash
python3 parallelcmd.py init [options] <command ...> [ ::: <args ...> ]* [ :::: <argfile ...> ]*
```

Options:
- `-a, --append` append to existing table instead of recreating
- `-f, --force` drop the existing `parjob` table and recreate it
- `--check_dup` skip commands that already exist
- `--zip` pair argument lists element-by-element instead of Cartesian product
- `-v, --verbose`

### `exec`

Execute queued jobs in parallel.

```bash
python3 parallelcmd.py exec [options]
```

Options:
- `-j, --nworkers <n>` number of workers (default: `4`)
- `--id <id ...>` run only these specific job IDs
- `--progress` show aggregate progress line
- `--bar` show a visual ASCII progress bar (alternative to `--progress`)
- `--eta` append estimated time remaining to the progress or bar line (use with `--progress` or `--bar`)
- `--dashboard` one live line per worker showing its latest output, redrawn in place
- `--dryrun` print commands without running; jobs are marked done (exit 0), so run `reset --all` before a real run
- `-v, --verbose`
- `--timeskip <sec>` print at most one output line per `<sec>` seconds; other lines are **discarded** from the terminal (still saved with `--output-dir`)
- `--randomorder` fetch pending jobs in random order
- `--descorder` fetch pending jobs in descending `Seq` order (newest first); cannot be combined with `--randomorder`
- `--prefix <cmd>` prefix each command; supports shell env var assignments (example: `srun -N1 -n1`, `NP=8`)
- `--max_jobs <n>` max jobs per worker
- `--delay <sec>` sleep this many seconds before starting each job (default: `0`); also used as the upper bound for the initial per-worker random stagger
- `--wait <sec>` when no job is available, wait this many seconds and retry instead of exiting (useful when another process is still adding jobs)
- `--timeout <sec>` kill a task and move to the next if it runs longer than this many seconds; timed-out jobs are recorded with exit code `124`
- `--kill-poll <sec>` how often to check the DB for `kill` requests (default: `2`; `0` disables)
- `--retries <n>` retry a failed job up to N times before marking it failed (default: `0`; timed-out and killed jobs are never retried)
- `--halt <n>` after N failures, each worker finishes its current job and exits; remaining jobs stay pending
- `--output-dir <dir>` also save each job's output (stdout and stderr) to `<dir>/<seq>.out`; the file is appended to, so retries and reruns accumulate
- `--quiet` suppress per-job output lines (useful with `--progress` or `--bar`)
- `--tag` prefix each output line with the full command instead of the seq ID
- `--hook <file>` Python plugin file; see [Hooks](#hooks) below

Output: each line a job prints is shown as `<seq>: <line>` (or `<command>: <line>` with `--tag`). stderr is merged into stdout. Lines from different jobs interleave as they arrive.

### `run`

Initialize and execute in one step (`init` + `exec`).

```bash
python3 parallelcmd.py run [options] <command ...> [ ::: <args ...> ]* [ :::: <argfile ...> ]*
```

`run` fails if the queue already exists; add `-a` to append or `-f` to start over.

Common options include:
- init side: `--append`, `-f/--force`, `--check_dup`, `--zip`
- exec side: `-j/--nworkers`, `--id`, `--progress`, `--bar`, `--eta`, `--dashboard`, `--dryrun`, `--randomorder`, `--descorder`, `--prefix`, `--max_jobs`, `--delay`, `--wait`, `--timeout`, `--kill-poll`, `--retries`, `--halt`, `--output-dir`, `--quiet`, `--tag`, `--hook`

### `check`

Inspect queue summary or list all rows.

```bash
python3 parallelcmd.py check [options]
```

Options:
- `-l, --list` list all matching rows instead of the summary
- `--nonzero` filter to only jobs with non-zero exit value
- `--running` filter to only currently running jobs
- `--where <sql>` arbitrary SQL `WHERE` clause
- `--like <pattern>` filter by `Command LIKE <pattern>`
- `--id <id ...>` filter by specific job IDs

If several filters are given: `--running` overrides everything else; otherwise `--id` wins over `--like`, which wins over `--where`, and `--nonzero` is applied on top.

Summary columns: `Total | Pending Running Killing | Success Failed Error`. `Error` counts negative exit values, e.g. killed jobs.

### `reset`

Reset selected jobs to pending so the next `exec` runs them again, or mark them with a fixed exit value so they are skipped.

```bash
python3 parallelcmd.py reset [--all | --nonzero | --like <pattern> | --id <id ...> | --where <sql>] [--done | --exitval <n>]
```

Selection:
- *(default)* all jobs with `Exitval <> 0`: failed, timed out, errored, **and in-progress**. Pending jobs are not matched. Don't use the default while an `exec` is running.
- `-a, --all` all jobs
- `--nonzero` only jobs with a positive exit value (applied on top of `--where`, `--like` or `--all`)
- `--where <sql>` arbitrary SQL `WHERE` clause
- `--like <pattern>` filter by `Command LIKE <pattern>`
- `--id <id ...>` filter by specific job IDs

These filters are not combined: if more than one is given, `--id` wins over `--all`, which wins over `--like`, which wins over `--where`.

Action:
- *(default)* back to pending: `Starttime`, `Hostname`, `PID`, `JobRuntime` and `Exitval` are set to `NULL`
- `--done` mark as done (`Exitval = 0`) without rerunning
- `--exitval <n>` mark with `Exitval = n` (e.g. `124` to treat a job as timed out)

Other:
- `-y, --yes` skip confirmation prompt

Matching rows are listed and a confirmation prompt is shown (skipped with `-y`).

`--done` and `--exitval` change only `Exitval`; the host, PID and runtime of the last run are kept. They always skip running jobs (including jobs with a pending `kill` request), since the worker would overwrite the value when the job finishes. `kill` the job first if needed.

Examples:

```bash
# rerun everything that failed
python3 parallelcmd.py reset -y

# rerun only timed-out jobs
python3 parallelcmd.py reset --where "Exitval = 124"

# a job failed for a known, harmless reason: accept it without rerunning
python3 parallelcmd.py reset --done --id 17 42

# skip a whole group of jobs that is no longer needed
python3 parallelcmd.py reset --done --like '%old_config%' -y
```

### `kill`

Stop running jobs, on any host.

```bash
python3 parallelcmd.py kill [--all | --id <id ...> | --where <sql> | --like <pattern> | --host <hostname>] [-s SIG] [--reset]
```

Options:
- `-a, --all` kill all running jobs
- `--id <id ...>`, `--where <sql>`, `--like <pattern>`, `--host <hostname>` select jobs (combined with AND; only running jobs are ever selected)
- `-s, --signal <sig>` signal name or number (default: `TERM`)
- `--reset` requeue killed jobs (`Exitval = NULL`) instead of recording `-signum`
- `--wait <sec>` how long to wait for the owning `exec` to act (default: `10`; `0` = send the request and return)
- `-y, --yes` skip confirmation prompt

Matching rows are listed and a confirmation prompt is shown (skipped with `-y`). At least one selection option is required.

How it works: `kill` does not signal PIDs directly, since a PID is only meaningful on its own host. It writes a kill request into the job's `Exitval` (`-1000 - signum`, or `-1100 - signum` with `--reset`). Every `exec` polls for requests on jobs it is running itself (see `--kill-poll`), kills the job's process group, and records `Exitval = -signum` (e.g. `-15`) or requeues it. Killed jobs are not retried and do not count toward `--halt`.

If the owning `exec` does not respond within `--wait`:
- **same host as `kill`** (orphan, e.g. its `exec` was killed): `kill` signals the process group itself, after checking the PID was not reused, and records the result.
- **other host**: the request stays pending and `kill` prints an `ssh <host> kill ...` hint. Clear leftover requests with `reset --where "Exitval BETWEEN -1164 AND -1001" -y`.

With `--reset` and `exec` still running, the requeued job is picked up again right away.

### `delete`

Delete selected jobs.

```bash
python3 parallelcmd.py delete [options]
```

Options:
- `-a, --all` delete all jobs
- `--like <pattern>` filter by SQL LIKE pattern on command text
- `--id <id ...>` filter by job ID(s)
- `-y, --yes` skip confirmation prompt

With no filter, deletes jobs with `Exitval <> 0` (same default as `reset`). Prompts for confirmation before deleting rows (skipped with `-y`).

### `update`

Find/replace command text for selected jobs.

```bash
python3 parallelcmd.py update [options]
```

Options:
- `--replace "old,new"` find and replace text pair (comma-separated)
- `--like <pattern>` filter by SQL LIKE pattern on command text
- `--id <id ...>` filter by job ID(s)
- `-y, --yes` skip confirmation prompt

With no filter, all jobs are updated. Prompts for confirmation before updating rows (skipped with `-y`).

> **Note:** if the replacement text starts with `--`, use the `=` form to prevent argparse from treating it as a flag:
> ```bash
> python3 parallelcmd.py update --replace='--old-flag,--new-flag'
> ```

### `diagnose`

Inspect the health of the SQLite database — useful when jobs appear stuck or the DB seems unresponsive.

```bash
python3 parallelcmd.py diagnose [--stale SECONDS]
```

Reports:
1. Job counts by state (pending / running / killing / success / failed / error)
2. In-progress jobs (`Exitval = -1000`, or a pending kill request) with age in seconds; flags any older than `--stale` (default: `3600`) as potentially stale
3. DB file sizes (`.sqlite`, `.sqlite-wal`, `.sqlite-shm`); warns if WAL exceeds 10 MB
4. Exclusive lock probe — attempts `BEGIN EXCLUSIVE` with a 2-second timeout
5. Open file handles via `lsof`

To recover stale in-progress jobs after a crash (this range also covers pending `kill` requests):

```bash
python3 parallelcmd.py reset --where "Exitval BETWEEN -1164 AND -1000" -y
```

## Hooks

`--hook <file>` loads a Python file that can inspect each job before and/or after it runs. Define either or both functions:

```python
def on_before_task(taskid, cmd):
    # called after --delay sleep, before the subprocess launches
    # return False → requeue this job to pending and stop this worker
    return True

def on_after_task(taskid, cmd, exitval, runtime):
    # called after the exit value is written to the DB
    # exitval: 124 on timeout, -signum if stopped by `kill`
    # return False → stop this worker (other workers keep running)
    return True
```

- Either function can be omitted — only the defined ones are called.
- Exceptions inside a hook are logged and treated as `True` (continue).
- Returning `False` stops only the calling worker; other workers are unaffected.

Example hook files are in the `hooks/` directory:

| File | Purpose |
|---|---|
| `hooks/my_slurm_hook.py` | Stop workers when SLURM remaining time drops below 1 hour |
| `hooks/my_pbs_hook.py` | Same for PBS/Torque (`qstat`) |

```bash
python3 parallelcmd.py exec -j 4 --hook=hooks/my_slurm_hook.py
```

Edit `CHECK_TIMELEFT` at the top of the hook file to adjust the threshold.

## Global options

- `--db <name>` SQLite DB basename; the file on disk is `<name>.sqlite` (a name already ending in `.sqlite` is used as is). Default: `$PARDB`, else `pardb`.
- `--db_retries <n>` max retries when SQLite is locked (default: `10`)
- `--log_level {debug,info}` logging level (default: `info`)

Global options must come **before** the subcommand: `parallelcmd.py --db jobs exec`, not `parallelcmd.py exec --db jobs`.

## Useful examples

Pipe arguments from stdin (auto-detected when no `:::` or `::::` is given):

```bash
cat cases.txt | python3 parallelcmd.py -j 4 "bash run.sh {}"
seq 10 | python3 parallelcmd.py "echo {}"
```

Pipe stdin explicitly with `:::: -` (combinable with other arg lists):

```bash
cat cases.txt | python3 parallelcmd.py run "bash run.sh {} {}" :::: - ::: seed1 seed2
```

Run scripts from values in a file:

```bash
python3 parallelcmd.py init "bash run_case.sh {}" :::: cases.txt
python3 parallelcmd.py exec -j 8
```

Use a custom DB file:

```bash
python3 parallelcmd.py --db jobs init "echo {}" ::: x y z
python3 parallelcmd.py --db jobs exec -j 2
```

Kill tasks that exceed a time limit and continue to the next job:

```bash
python3 parallelcmd.py exec -j 4 --timeout 300
```

Timed-out jobs are recorded with exit code `124`. Find them with:

```bash
python3 parallelcmd.py check -l --where "Exitval = 124"
```

Reset timed-out jobs to retry with a longer timeout:

```bash
python3 parallelcmd.py reset --where "Exitval = 124"
python3 parallelcmd.py exec -j 4 --timeout 600
```

Keep workers alive while another process appends jobs later:

```bash
python3 parallelcmd.py exec -j 4 --wait 10
python3 parallelcmd.py init -a "echo {}" ::: later1 later2
```

Retry failed jobs only:

```bash
python3 parallelcmd.py reset
python3 parallelcmd.py exec -j 4
```

Overwrite the queue with a new set of jobs (drop and recreate):

```bash
python3 parallelcmd.py init -f "echo {}" ::: x y z
python3 parallelcmd.py exec -j 4
```

## Multiple workers and nodes

Several `exec` processes can share one queue, on the same machine or on different nodes, as long as they all see the DB file (e.g. on a shared filesystem). Each job is claimed atomically, so no job runs twice. This lets you scale up by starting more `exec`s at any time, even while jobs are being added.

```bash
# in a SLURM allocation: one exec per node, 8 workers each
srun -N $SLURM_NNODES -n $SLURM_NNODES python3 parallelcmd.py exec -j 8 --wait 30
```

- `Hostname` and `PID` record where each job runs: `check -l --running`.
- `{%}` is the worker slot *within one exec*, so it is safe for per-node GPU binding.
- `kill` works across nodes: it asks the `exec` that owns the job to kill it (`kill --host <node>` targets one node).
- `--delay` staggers each worker's first job, which helps when many workers start at once.
- More processes means more SQLite lock contention; a busy DB retries automatically (`--db_retries`). If you see persistent `database is locked` errors, use fewer `exec` processes with more workers each, or run `diagnose`.

## Notes

- Job output is streamed to stdout while running.
- Queue state is persisted in SQLite, so you can stop and resume workflows.
- `reset`, `delete`, `update`, and `kill` prompt for confirmation by default; pass `-y` to skip.
- With `--wait`, workers poll for newly appended jobs instead of exiting as soon as the queue is empty.

## Aliases

Add these to `~/.bashrc` or `~/.zshrc` to avoid typing the full command each time.
Assumes `parallelcmd.py` is on your `PATH`.

```bash
# parallelcmd aliases
alias pc='parallelcmd.py'

# init
alias pci='parallelcmd.py init'
alias pcia='parallelcmd.py init --append'
alias pcif='parallelcmd.py init --force'

# exec
alias pce='parallelcmd.py exec'
alias pcer='parallelcmd.py exec --randomorder'
alias pced='parallelcmd.py exec --descorder'
alias pcep='parallelcmd.py exec --progress'

# check
alias pck='parallelcmd.py check'
alias pckl='parallelcmd.py check -l'
alias pckf='parallelcmd.py check -l --nonzero'

# reset / delete / update / kill
alias pcr='parallelcmd.py reset'
alias pcra='parallelcmd.py reset --all'
alias pcrf='parallelcmd.py reset --nonzero'
alias pcdone='parallelcmd.py reset --done --id'
alias pcd='parallelcmd.py delete'
alias pcda='parallelcmd.py delete --all'
alias pcu='parallelcmd.py update'
alias pckill='parallelcmd.py kill'

# reset timed-out jobs
alias pctimeout='parallelcmd.py reset --where "Exitval = 124"'

# exec with N workers and progress  (usage: pcej 8)
pcej() { parallelcmd.py exec -j "$@"; }

# run (init + exec) with common worker counts and progress
pcj4()  { parallelcmd.py run -j 4 "$@"; }
pcj8()  { parallelcmd.py run -j 8 "$@"; }
pcj16() { parallelcmd.py run -j 16 "$@"; }
```

## Troubleshooting

- **`database is locked`**
	- Usually temporary when multiple workers/processes access SQLite; it is retried automatically (`--db_retries`).
	- If it persists, run `python3 parallelcmd.py diagnose`, or use fewer `exec` processes with more workers each.

- **`exec` hangs with no output, even for trivial jobs**
	- Your `python3` is probably older than 3.8 (check `python3 --version`); on those versions `exec` hangs silently. See [Requirements](#requirements).

- **`KeyError` or `IndexError` from `init`**
	- The command contains literal braces (write `{{` and `}}`), or uses more `{}` than there are argument lists (use `{0}`, `{1}`).

- **`parjob table already exists`**
	- Use `init -a` / `run -a` to add jobs, or `-f` to discard the old queue.

- **No jobs are executed**
	- Check queue state: `python3 parallelcmd.py check -l`.
	- If jobs are stuck in-progress (their `exec` died), reset them: `python3 parallelcmd.py reset` (see [`diagnose`](#diagnose)).
	- Completed jobs (exit `0`) are never rerun; use `reset --all` to rerun everything.

- **Workers exit before later jobs are appended**
	- Start `exec` with `--wait <seconds>` so workers keep polling.
	- Append work with `init -a ...` from another process or terminal.

- **Unexpected shell behavior / quoting issues**
	- Commands are executed through `bash -c`.
	- Wrap complex commands in quotes and test one command manually before `init`.

- **Stop workers based on SLURM/PBS remaining time**
	- Use `--hook=hooks/my_slurm_hook.py` (or `my_pbs_hook.py`).
	- Must be run inside an allocation where `SLURM_JOB_ID` / `PBS_JOBID` is set.

- **Some jobs have exit code `124`**
	- These jobs were killed by `--timeout`.
	- Reset and retry them: `python3 parallelcmd.py reset --where "Exitval = 124"`, then re-run `exec` with a larger `--timeout` or without it.

- **`update --replace` does not parse as expected**
	- Use exactly one comma-separated pair: `--replace "old,new"`.
	- If your text contains commas, run multiple updates with simpler replacement pairs.

- **Argument file (`::::`) seems ignored**
	- Ensure one argument per line.
	- Blank lines and lines starting with `#` are intentionally skipped.

## Comparison with GNU Parallel

| Feature | GNU Parallel | parallelcmd |
|---|---|---|
| **Input: inline list** | `:::` | `:::` |
| **Input: file** | `::::` | `::::` |
| **Input: stdin (auto)** | pipe or `-` | pipe (auto-detected when no `:::`) |
| **Input: stdin (explicit)** | `:::: -` | `:::: -` |
| **Input: multiple lists** | Cartesian product | Cartesian product |
| **Input: linked/paired lists** | `--link` | `--zip` |
| **Column split** | `--colsep REGEX` | — |
| **Null delimiter** | `-0` | — |
| **Stop at sentinel** | `-E VALUE` | — |
| **Skip empty lines** | `--no-run-if-empty` | — |
| **Arg substitution: full** | `{}` | `{}` |
| **Arg substitution: no ext** | `{.}` | — |
| **Arg substitution: basename** | `{/}` | — |
| **Arg substitution: dirname** | `{//}` | — |
| **Arg substitution: job #** | `{#}` | `{#}` |
| **Arg substitution: slot #** | `{%}` | `{%}` |
| **Positional substitution** | `{1}`, `{2}`, … | `{0}`, `{1}`, … |
| **Workers** | `-j N` | `-j N` |
| **Load-based throttle** | `--load`, `--noswap`, `--memfree` | — |
| **Nice/priority** | `--nice` | — |
| **Startup delay** | `--delay SEC` | `--delay SEC` |
| **Progress bar** | `--progress`, `--eta`, `--bar` | `--progress`, `--bar`, `--eta`, `--dashboard` |
| **Job log** | `--joblog FILE` | SQLite DB (always persisted) |
| **Resume incomplete batch** | `--resume` (via joblog) | re-run `exec` (auto, SQLite state) |
| **Retry failed only** | `--resume-failed` | `reset --nonzero` + `exec` |
| **Retry N times** | `--retries N` | `--retries N` |
| **Skip duplicates** | — | `--check_dup` |
| **Output order** | `-k` / `--keep-order` | — (streamed as-is) |
| **Tag output** | `--tag`, `--tagstring` | `--tag` |
| **Save results to dir** | `--results DIR` | `--output-dir DIR` |
| **Immediate streaming** | `--ungroup` | always streamed |
| **Line buffering** | `--linebuffer` | — |
| **Timeout** | `--timeout DURATION` | `--timeout SEC` |
| **Exit code for timeout** | 124 | 124 |
| **Halt on failure** | `--halt soon/now,fail=N` | `--halt N` |
| **Custom kill signal** | `--termseq` | `kill -s SIG` (on request, not on timeout) |
| **Kill running jobs** | Ctrl-C the parallel process | `kill` (from any host, by ID/filter/host) |
| **Dry-run** | `--dry-run` | `--dryrun` |
| **Verbose / print cmd** | `--verbose` | `-v` / `--verbose` |
| **Random order** | `--shuf` | `--randomorder` |
| **Reverse order** | — | `--descorder` |
| **Interactive confirm** | `--interactive` | — |
| **Command prefix** | `--` (shell) | `--prefix CMD` |
| **SLURM/PBS time-limit hook** | — | `--hook FILE` (`hooks/my_slurm_hook.py`) |
| **Before/after job hooks** | — | `--hook FILE` (`on_before_task`, `on_after_task`) |
| **Remote execution** | `--sshlogin`, `--slf`, `--trc` | — |
| **Distributed file sync** | `--transfer`, `--return`, `--cleanup` | — |
| **Pipe/streaming mode** | `--pipe`, `--block`, `--pipepart` | — |
| **Semaphore mode** | `sem` / `--semaphore` | — |
| **tmux integration** | `--tmux` | — |
| **Multiple queues** | separate invocations | `--db NAME` (named SQLite files) |
| **Inspect queue** | `--joblog` + external tools | `check`, `check -l`, `--where`, `--like` |
| **Edit queued commands** | — | `update --replace` |
| **Delete specific jobs** | — | `delete --id`, `delete --like` |
| **Reset specific jobs** | — | `reset --id`, `reset --where` |
| **Mark jobs done without running** | — | `reset --done` |
| **Wait for new jobs** | — | `--wait SEC` (keep workers polling) |
| **Max jobs per worker** | — | `--max_jobs N` |
| **External dependencies** | none (Perl) | none (Python stdlib only) |
| **Persistent state** | optional (joblog file) | always (SQLite) |

GNU Parallel is broader for one-shot parallel execution — especially argument substitution, remote/distributed runs, pipe streaming, and output formatting. `parallelcmd` trades those for a persistent job queue with first-class management (inspect, edit, delete, reset by SQL filter) and native SLURM time-limit awareness, making it better suited for long-running experiment pipelines where you need to stop, resume, and selectively retry jobs across sessions.

## FAQ

- **How do I resume after interruption?**
	- Just run `python3 parallelcmd.py exec -j 4` again.
	- Completed jobs (exit code `0`) stay done; pending jobs continue.

- **How do I retry only failed jobs?**
	- Failed jobs are those with non-zero exit values.
	- Run `python3 parallelcmd.py reset` (default filter resets jobs with `Exitval <> 0`), then run `exec` again.
	- Use `--nonzero` to be explicit: `python3 parallelcmd.py reset --nonzero`.

- **What does exit code `124` mean?**
	- The job was killed by `--timeout`. This matches the GNU `timeout` exit code convention.
	- Reset and rerun: `python3 parallelcmd.py reset --where "Exitval = 124"`, then `exec` with a longer `--timeout`.

- **Can I have multiple queues?**
	- Yes. Use different database basenames with `--db`.
	- Example: `python3 parallelcmd.py --db exp1 init ...` then `exec` using the same `--db`.

- **Is it safe to run two `exec` commands on the same DB?**
	- Yes. Jobs are claimed atomically, so each job runs once. See [Multiple workers and nodes](#multiple-workers-and-nodes).
	- The cost is more SQLite lock contention as the number of processes grows.

- **How do I stop a job that is running?**
	- `python3 parallelcmd.py kill --id <seq>`. See [`kill`](#kill).

- **How do I skip a job without running it?**
	- `python3 parallelcmd.py reset --done --id <seq>` marks it as done.

- **Can I inspect/edit queued commands before running?**
	- Inspect: `python3 parallelcmd.py check --list`
	- Bulk edit text: `python3 parallelcmd.py update --replace "old,new" --like "%pattern%"`
	- Remove unwanted rows: `python3 parallelcmd.py delete --id 12 13 14`
