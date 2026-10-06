# FAQ

### How do I resume after interruption?

Just run `python3 parallelcmd.py exec -j 4` again. Completed jobs (exit code `0`) stay done; pending jobs continue. Jobs that were running when the interruption happened stay in-progress; reset them first (see [`diagnose`](usage/managing.md#diagnose)).

### How do I retry only failed jobs?

Run `python3 parallelcmd.py reset --nonzero`, then run `exec` again. A plain `reset` uses the default filter `Exitval <> 0`, which also includes in-progress and killed jobs.

### What does exit code `124` mean?

The job was killed by `--timeout`. This matches the GNU `timeout` exit code convention. Reset and rerun with `python3 parallelcmd.py reset --where "Exitval = 124"`, then `exec` with a longer `--timeout`.

### Can I have multiple queues?

Yes. Use different database basenames with `--db`, e.g. `python3 parallelcmd.py --db exp1 init ...`, then `exec` with the same `--db`.

### Is it safe to run two `exec` commands on the same DB?

Yes. Jobs are claimed atomically, so each job runs once. See [Multiple workers and nodes](usage/multinode.md). The cost is more SQLite lock contention as the number of processes grows.

### How do I stop a job that is running?

`python3 parallelcmd.py kill --id <seq>`. See [Stopping jobs](usage/kill.md).

### How do I skip a job without running it?

`python3 parallelcmd.py reset --done --id <seq>` marks it as done.

### Can I inspect/edit queued commands before running?

- Inspect: `python3 parallelcmd.py check --list`
- Bulk edit text: `python3 parallelcmd.py update --replace "old,new" --like "%pattern%"`
- Remove unwanted rows: `python3 parallelcmd.py delete --id 12 13 14`
