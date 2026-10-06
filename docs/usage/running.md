# Running jobs

## Global options

These apply to every subcommand and must come **before** it: `parallelcmd.py --db jobs exec`, not `parallelcmd.py exec --db jobs`.

- `--db <name>` SQLite DB basename; the file on disk is `<name>.sqlite` (a name already ending in `.sqlite` is used as is). Default: `$PARDB`, else `pardb`.
- `--db_retries <n>` max retries when SQLite is locked (default: `10`)
- `--log_level {debug,info}` logging level (default: `info`)

## `init`

Initialize the job queue, or append to an existing one.

```bash
python3 parallelcmd.py init [options] <command ...> [ ::: <args ...> ]* [ :::: <argfile ...> ]*
```

Options:

- `-a, --append` append to the existing queue
- `-f, --force` drop the existing `parjob` table and recreate it
- `--check_dup` skip commands that already exist
- `--zip` pair argument lists element-by-element instead of Cartesian product
- `-v, --verbose`

Without `-a` or `-f`, `init` fails if the queue already exists. `-a` and `-f` cannot be combined.

## `exec`

Execute queued jobs in parallel.

```bash
python3 parallelcmd.py exec [options]
```

**Workers and job selection**

- `-j, --nworkers <n>` number of workers (default: `4`)
- `--id <id ...>` run only these specific job IDs
- `--randomorder` fetch pending jobs in random order
- `--descorder` fetch pending jobs in descending `Seq` order (newest first); cannot be combined with `--randomorder`
- `--max_jobs <n>` max jobs per worker
- `--wait <sec>` when no job is available, wait this many seconds and retry instead of exiting (useful when another process is still adding jobs)
- `--delay <sec>` sleep this many seconds before starting each job (default: `0`); also used as the upper bound for the initial per-worker random stagger
- `--prefix <cmd>` prefix each command; supports shell env var assignments (example: `srun -N1 -n1`, `NP=8`)
- `--dryrun` print commands without running; jobs are marked done (exit 0), so run `reset --all` before a real run

**Failure handling**

- `--timeout <sec>` kill a task and move to the next if it runs longer than this many seconds; timed-out jobs are recorded with exit code `124`
- `--retries <n>` retry a failed job up to N times before marking it failed (default: `0`; timed-out and killed jobs are never retried)
- `--halt <n>` after N failures, each worker finishes its current job and exits; remaining jobs stay pending
- `--kill-poll <sec>` how often to check the DB for [`kill`](kill.md) requests (default: `2`; `0` disables)
- `--hook <file>` Python plugin file; see [Hooks](hooks.md)

**Output**

- `--progress` show aggregate progress line
- `--bar` show a visual ASCII progress bar (alternative to `--progress`)
- `--eta` append estimated time remaining to the progress or bar line (use with `--progress` or `--bar`)
- `--dashboard` one live line per worker showing its latest output, redrawn in place
- `--quiet` suppress per-job output lines (useful with `--progress` or `--bar`)
- `--tag` prefix each output line with the full command instead of the seq ID
- `--timeskip <sec>` print at most one output line per `<sec>` seconds; other lines are **discarded** from the terminal (still saved with `--output-dir`)
- `--output-dir <dir>` also save each job's output (stdout and stderr) to `<dir>/<seq>.out`; the file is appended to, so retries and reruns accumulate
- `-v, --verbose`

### Output format

Each line a job prints is shown as `<seq>: <line>` (or `<command>: <line>` with `--tag`). stderr is merged into stdout. Lines from different jobs interleave as they arrive.

Commands are executed through `bash -c`.

## `run`

Initialize and execute in one step (`init` + `exec`).

```bash
python3 parallelcmd.py run [options] <command ...> [ ::: <args ...> ]* [ :::: <argfile ...> ]*
```

`run` accepts all `init` and `exec` options. Like `init`, it fails if the queue already exists; add `-a` to append or `-f` to start over.

If no subcommand is given, `run` is assumed:

```bash
python3 parallelcmd.py -j 4 "echo {}" ::: a b c
```
