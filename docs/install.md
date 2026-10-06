# Installation

Download the single script and make it executable — no pip or dependencies required.

```bash
# wget
wget https://raw.githubusercontent.com/jychoi-hpc/parallelcmd/main/parallelcmd.py
chmod +x parallelcmd.py

# curl
curl -O https://raw.githubusercontent.com/jychoi-hpc/parallelcmd/main/parallelcmd.py
chmod +x parallelcmd.py
```

You can then run it directly:

```bash
./parallelcmd.py --help
```

Or place it somewhere on your `PATH` (e.g. `~/.local/bin/`) to use it as `parallelcmd.py` from any directory.

## Requirements

- Python 3.8+
- Standard library only (no external Python dependencies)

!!! warning "Check your Python version"
    Some systems ship an older `python3` (e.g. Python 3.6 on OLCF Frontier login nodes). On those versions `exec` hangs silently. Check with `python3 --version`, and use a newer interpreter if needed, e.g. `python3.11 parallelcmd.py ...` or `module load cray-python`.
