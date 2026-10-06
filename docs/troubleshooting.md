# Troubleshooting

### `database is locked`

- Usually temporary when multiple workers/processes access SQLite; it is retried automatically (`--db_retries`).
- If it persists, run `python3 parallelcmd.py diagnose`, or use fewer `exec` processes with more workers each.

### `exec` hangs with no output, even for trivial jobs

- Your `python3` is probably older than 3.8 (check `python3 --version`); on those versions `exec` hangs silently. See [Requirements](install.md#requirements).

### `KeyError` or `IndexError` from `init`

- The command contains literal braces (write `{{` and `}}`), or uses more `{}` than there are argument lists (use `{0}`, `{1}`). See [Placeholders](concepts.md#placeholders).

### `parjob table already exists`

- Use `init -a` / `run -a` to add jobs, or `-f` to discard the old queue.

### No jobs are executed

- Check queue state: `python3 parallelcmd.py check -l`.
- If jobs are stuck in-progress (their `exec` died), reset them: `python3 parallelcmd.py reset` (see [`diagnose`](usage/managing.md#diagnose)).
- Completed jobs (exit `0`) are never rerun; use `reset --all` to rerun everything.

### Workers exit before later jobs are appended

- Start `exec` with `--wait <seconds>` so workers keep polling.
- Append work with `init -a ...` from another process or terminal.

### Unexpected shell behavior / quoting issues

- Commands are executed through `bash -c`.
- Wrap complex commands in quotes and test one command manually before `init`.

### Stop workers based on SLURM/PBS remaining time

- Use `--hook=hooks/my_slurm_hook.py` (or `my_pbs_hook.py`). See [Hooks](usage/hooks.md).
- Must be run inside an allocation where `SLURM_JOB_ID` / `PBS_JOBID` is set.

### Some jobs have exit code `124`

- These jobs were killed by `--timeout`.
- Reset and retry them: `python3 parallelcmd.py reset --where "Exitval = 124"`, then re-run `exec` with a larger `--timeout` or without it.

### `update --replace` does not parse as expected

- Use exactly one comma-separated pair: `--replace "old,new"`.
- If your text contains commas, run multiple updates with simpler replacement pairs.

### Argument file (`::::`) seems ignored

- Ensure one argument per line.
- Blank lines and lines starting with `#` are intentionally skipped.
