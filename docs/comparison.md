# Comparison with GNU Parallel

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
