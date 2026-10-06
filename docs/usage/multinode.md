# Multiple workers and nodes

Several `exec` processes can share one queue, on the same machine or on different nodes, as long as they all see the DB file (e.g. on a shared filesystem). Each job is claimed atomically, so no job runs twice. This lets you scale up by starting more `exec`s at any time, even while jobs are being added.

```bash
# in a SLURM allocation: one exec per node, 8 workers each
srun -N $SLURM_NNODES -n $SLURM_NNODES python3 parallelcmd.py exec -j 8 --wait 30
```

- `Hostname` and `PID` record where each job runs: `check -l --running`.
- `{%}` is the worker slot *within one exec*, so it is safe for per-node GPU binding.
- [`kill`](kill.md) works across nodes: it asks the `exec` that owns the job to kill it (`kill --host <node>` targets one node).
- `--delay` staggers each worker's first job, which helps when many workers start at once (it also adds that delay before every job).
- `--wait` keeps workers polling for jobs appended later with `init -a`.

!!! tip "Lock contention"
    More processes means more SQLite lock contention; a busy DB retries automatically (`--db_retries`). If you see persistent `database is locked` errors, use fewer `exec` processes with more workers each, or run [`diagnose`](managing.md#diagnose).

To stop workers before the allocation ends, see the SLURM/PBS time-limit [hooks](hooks.md).
