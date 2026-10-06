# Stopping jobs

## `kill`

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

At least one selection option is required. Matching rows are listed and a confirmation prompt is shown.

Examples:

```bash
# SIGTERM two jobs; they are recorded as Exitval -15
python3 parallelcmd.py kill --id 12 15 -y

# everything running on one node, with SIGKILL
python3 parallelcmd.py kill --host frontier01234 -s KILL

# kill and put back in the queue
python3 parallelcmd.py kill --like '%big%' --reset
```

## How it works

A PID is only meaningful on the host that started it, and `kill` may be run from a different node (or a login node). So `kill` does not signal PIDs directly:

1. It writes a kill request into the job's `Exitval`: `-1000 - signum`, or `-1100 - signum` with `--reset`.
2. Every `exec` polls for requests on jobs it is running itself (every `--kill-poll` seconds), and kills the job's process group.
3. The job is recorded as `Exitval = -signum` (e.g. `-15`), or requeued with `--reset`. Killed jobs are not retried and do not count toward `--halt`.

If the owning `exec` does not respond within `--wait`:

- **Same host as `kill`** (an orphan, e.g. its `exec` was killed): `kill` signals the process group itself, after checking the PID was not reused, and records the result.
- **Other host**: the request stays pending and `kill` prints an `ssh <host> kill ...` hint. Clear leftover requests with:

    ```bash
    python3 parallelcmd.py reset --where "Exitval BETWEEN -1164 AND -1001" -y
    ```

!!! note
    With `--reset` and `exec` still running, the requeued job is picked up again right away.

## Timeouts

To stop jobs automatically instead, use `exec --timeout <sec>`. Timed-out jobs are killed with `SIGKILL` and recorded with exit code `124`:

```bash
python3 parallelcmd.py exec -j 4 --timeout 300
python3 parallelcmd.py check -l --where "Exitval = 124"
```
