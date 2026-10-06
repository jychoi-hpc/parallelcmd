# Hooks

`exec --hook <file>` loads a Python file that can inspect each job before and/or after it runs. Define either or both functions:

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

## Example hooks

Example hook files are in the [`hooks/`](https://github.com/jychoi-hpc/parallelcmd/tree/main/hooks) directory:

| File | Purpose |
|---|---|
| `hooks/my_slurm_hook.py` | Stop workers when SLURM remaining time drops below 1 hour |
| `hooks/my_pbs_hook.py` | Same for PBS/Torque (`qstat`) |

```bash
python3 parallelcmd.py exec -j 4 --hook=hooks/my_slurm_hook.py
```

Edit `CHECK_TIMELEFT` at the top of the hook file to adjust the threshold. These hooks must run inside an allocation where `SLURM_JOB_ID` / `PBS_JOBID` is set.
