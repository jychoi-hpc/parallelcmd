# Managing the queue

`reset`, `delete`, `update`, and `kill` list the matching rows and ask for confirmation; pass `-y` to skip the prompt.

## `check`

Inspect the queue summary or list rows.

```bash
python3 parallelcmd.py check [options]
```

Options:

- `-l, --list` list all matching rows instead of the summary
- `--nonzero` filter to only jobs with a positive exit value (failed or timed out)
- `--running` filter to only currently running jobs
- `--where <sql>` arbitrary SQL `WHERE` clause
- `--like <pattern>` filter by `Command LIKE <pattern>`
- `--id <id ...>` filter by specific job IDs

If several filters are given: `--running` overrides everything else; otherwise `--id` wins over `--like`, which wins over `--where`, and `--nonzero` is applied on top.

Summary columns: `Total | Pending Running Killing | Success Failed Error`. `Error` counts negative exit values, e.g. killed jobs. See [Job states](../concepts.md#job-states).

## `reset`

Reset selected jobs to pending so the next `exec` runs them again, or mark them with a fixed exit value so they are skipped.

```bash
python3 parallelcmd.py reset [--all | --nonzero | --like <pattern> | --id <id ...> | --where <sql>] [--done | --exitval <n>]
```

**Selection**

- *(default)* all jobs with `Exitval <> 0`: failed, timed out, errored, **and in-progress**. Pending jobs are not matched.
- `-a, --all` all jobs
- `--nonzero` only jobs with a positive exit value (applied on top of `--where`, `--like` or `--all`)
- `--where <sql>` arbitrary SQL `WHERE` clause
- `--like <pattern>` filter by `Command LIKE <pattern>`
- `--id <id ...>` filter by specific job IDs

These filters are not combined: if more than one is given, `--id` wins over `--all`, which wins over `--like`, which wins over `--where`.

!!! warning
    The default selection includes running jobs. Don't use it while an `exec` is running.

**Action**

- *(default)* back to pending: `Starttime`, `Hostname`, `PID`, `JobRuntime` and `Exitval` are set to `NULL`
- `--done` mark as done (`Exitval = 0`) without rerunning
- `--exitval <n>` mark with `Exitval = n` (e.g. `124` to treat a job as timed out)
- `-y, --yes` skip confirmation prompt

`--done` and `--exitval` change only `Exitval`; the host, PID and runtime of the last run are kept. They always skip running jobs (including jobs with a pending `kill` request), since the worker would overwrite the value when the job finishes. [`kill`](kill.md) the job first if needed.

Examples:

```bash
# rerun failed and timed-out jobs
python3 parallelcmd.py reset --nonzero -y

# rerun only timed-out jobs
python3 parallelcmd.py reset --where "Exitval = 124"

# a job failed for a known, harmless reason: accept it without rerunning
python3 parallelcmd.py reset --done --id 17 42

# skip a whole group of jobs that is no longer needed
python3 parallelcmd.py reset --done --like '%old_config%' -y
```

## `delete`

Delete selected jobs.

```bash
python3 parallelcmd.py delete [options]
```

Options:

- `-a, --all` delete all jobs
- `--like <pattern>` filter by SQL LIKE pattern on command text
- `--id <id ...>` filter by job ID(s)
- `-y, --yes` skip confirmation prompt

With no filter, deletes jobs with `Exitval <> 0` (same default as `reset`, so it includes in-progress jobs).

## `update`

Find/replace command text for selected jobs.

```bash
python3 parallelcmd.py update [options]
```

Options:

- `--replace "old,new"` find and replace text pair (comma-separated)
- `--like <pattern>` filter by SQL LIKE pattern on command text
- `--id <id ...>` filter by job ID(s)
- `-y, --yes` skip confirmation prompt

With no filter, all jobs are updated.

!!! note
    If the replacement text starts with `--`, use the `=` form to prevent argparse from treating it as a flag:

    ```bash
    python3 parallelcmd.py update --replace='--old-flag,--new-flag'
    ```

## `diagnose`

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
