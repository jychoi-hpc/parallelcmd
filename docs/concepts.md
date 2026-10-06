# Concepts

## Command model

`init` builds commands and stores them in a SQLite queue, `pardb.sqlite` by default. Use `--db <name>` to target `<name>.sqlite`, or set the `PARDB` environment variable. `init` (and `run`) refuse to touch an existing queue unless you pass `-a/--append` (add jobs) or `-f/--force` (drop and recreate).

- `:::` starts an inline argument list.
- `::::` starts an argument list loaded from a file (one value per line; empty lines and `#` comments are ignored).
- `:::: -` reads the argument list from stdin.
- Multiple lists are combined with Cartesian product (or paired with `--zip`).
- If the command has no argument placeholders, one `{}` per argument list is appended automatically (`init "echo" ::: a b ::: x` gives `echo a x`, `echo b x`). `{%}` and `{#}` don't count, so `"echo {%}" ::: a b` gives `echo {%} a`, `echo {%} b`.
- If no `:::` or `::::` separator is given and stdin is a pipe, stdin lines are used as the argument list automatically. If the command uses `{0}`, `{1}`, …, each line is split on whitespace into that many fields and the fields are paired as with `--zip` (the last field keeps any remaining text).

## Placeholders

| Placeholder | Meaning |
|---|---|
| `{}` | Current argument (positional, auto-assigned left to right) |
| `{0}`, `{1}`, … | Explicit positional argument from the Nth `:::` / `::::` list (0-indexed) |
| `{%}` | Worker slot number (0-indexed, stable for the lifetime of `exec`) |
| `{#}` | Job sequence number (the DB `Seq` of the job being run) |

`{}` and `{0}` / `{1}` can be mixed freely. `{}` takes the next list each time it appears, so a command using the same argument twice needs `{0}` (e.g. `"cp {0} {0}.bak"`). `{%}` and `{#}` are substituted at run time, not at `init` time, so the stored command retains the literal placeholder.

!!! warning "Literal braces must be doubled"
    Commands are expanded with Python `str.format`, so write `awk '{{print $1}}'` to get `awk '{print $1}'`. A single `{print $1}` fails with `KeyError`.

Cartesian product with implicit placeholders (6 jobs):

```bash
python3 parallelcmd.py init "python train.py --lr {} --seed {}" ::: 1e-3 1e-4 ::: 1 2 3
```

Explicit positional placeholders (same result, order made explicit):

```bash
python3 parallelcmd.py init "python train.py --lr {0} --seed {1}" ::: 1e-3 1e-4 ::: 1 2 3
```

GPU assignment via worker slot — each worker holds a fixed slot (0–3), so all its jobs run on the same GPU:

```bash
python3 parallelcmd.py run -j 4 "CUDA_VISIBLE_DEVICES={%} python train.py --lr {}" ::: 1e-3 1e-4 1e-5 1e-6
```

`--zip` pairs lists element-by-element instead of taking the Cartesian product. It stops at the shortest list when lengths differ.

```bash
# Without --zip: 4 jobs (a+x, a+y, b+x, b+y)
python3 parallelcmd.py init "cmd {0} {1}" ::: a b ::: x y

# With --zip: 2 jobs (a+x, b+y)
python3 parallelcmd.py init --zip "cmd {0} {1}" ::: a b ::: x y
```

## Job states

Each job is a row in the `parjob` table. Its state is stored in `Exitval`, which is what `--where` filters match on:

| State | `Exitval` |
|---|---|
| Pending | `NULL` |
| Running | `-1000` |
| Running, `kill` requested | `-1001` … `-1164` (see [Stopping jobs](usage/kill.md)) |
| Success | `0` |
| Failed | the command's exit code (`> 0`) |
| Timed out (`--timeout`) | `124` |
| Killed by `kill` | `-signum`, e.g. `-15` for `TERM` |

Other columns: `Seq` (job ID, used by `--id` and `{#}`), `Command`, `Starttime`, `Hostname` and `PID` (of the last run), `JobRuntime` (seconds).

Because state is persisted, you can stop and resume at any time: rerunning `exec` picks up the pending jobs, and completed jobs (exit `0`) stay done.
