# Examples

## Inputs

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

## Queues

Use a custom DB file:

```bash
python3 parallelcmd.py --db jobs init "echo {}" ::: x y z
python3 parallelcmd.py --db jobs exec -j 2
```

Keep workers alive while another process appends jobs later:

```bash
python3 parallelcmd.py exec -j 4 --wait 10
python3 parallelcmd.py init -a "echo {}" ::: later1 later2
```

Overwrite the queue with a new set of jobs (drop and recreate):

```bash
python3 parallelcmd.py init -f "echo {}" ::: x y z
python3 parallelcmd.py exec -j 4
```

## Retrying

Kill tasks that exceed a time limit and continue to the next job:

```bash
python3 parallelcmd.py exec -j 4 --timeout 300
```

Timed-out jobs are recorded with exit code `124`. Find them, then retry with a longer timeout:

```bash
python3 parallelcmd.py check -l --where "Exitval = 124"
python3 parallelcmd.py reset --where "Exitval = 124"
python3 parallelcmd.py exec -j 4 --timeout 600
```

Retry failed jobs only:

```bash
python3 parallelcmd.py reset --nonzero
python3 parallelcmd.py exec -j 4
```

## Shell aliases

Add these to `~/.bashrc` or `~/.zshrc` to avoid typing the full command each time. Assumes `parallelcmd.py` is on your `PATH`.

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

# exec with N workers  (usage: pcej 8)
pcej() { parallelcmd.py exec -j "$@"; }

# run (init + exec) with common worker counts
pcj4()  { parallelcmd.py run -j 4 "$@"; }
pcj8()  { parallelcmd.py run -j 8 "$@"; }
pcj16() { parallelcmd.py run -j 16 "$@"; }
```
