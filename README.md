# linpriv-audit

**Detection-only Linux privilege escalation audit toolkit.**

Scans a Linux host for common privilege escalation vectors, scores overall
security posture (0-100, A-F), and produces a report in text, HTML, and JSON.
Every check is read-only enumeration — the tool never executes exploit code
or modifies the target system.

```
 _     _       ____       _
| |   (_)_ __ |  _ \ _ __(_)_   __
| |   | | '_ \| |_) | '__| \ \ / /
| |___| | | | |  __/| |  | |\ V /
|_____|_|_| |_|_|   |_|  |_| \_/
```

## Install

**pip (recommended):**
```bash
pip install -e .
linpriv-audit --version
```

**Or without installing, just the dependencies:**
```bash
pip install -r requirements.txt
python3 main.py --version
```

**Docker** (also solves "I'm on Windows"):
```bash
docker build -t linpriv-audit .
docker run --rm -v "$PWD/report_output:/app/report_output" linpriv-audit
```
> Note: a container is namespace-isolated from the host, so a scan run this
> way audits the *container's* filesystem/config, not your host machine. For
> a real host audit on Windows, use WSL instead (see below) and run the
> toolkit directly inside it.

**Windows users:** this is a Linux tool — it reads `/etc/shadow`, `systemctl`,
`sudo -l`, cron files, etc., none of which exist on Windows. Running
`main.py` directly on Windows will now fail with a clear message (not a
crash) pointing you to WSL, Docker, or a VM. Fastest path:
```powershell
wsl --install
```
then run the toolkit inside the Ubuntu shell it installs, from your project
directory at `/mnt/c/Users/<you>/...`.

Requires Python 3.8+.

## Usage

```bash
# Full scan, full terminal UI
linpriv-audit                      # or: python3 main.py

# Run specific modules only
linpriv-audit --modules suid,permissions,cron

# Include INFO-level findings (full inventory, not just actionable issues)
linpriv-audit --show-info

# Custom output directory
linpriv-audit --out-dir /tmp/audit_results

# Exclude noisy paths from filesystem scans
linpriv-audit --exclude /mnt,/media,/nfs

# CI-friendly: exit non-zero if a HIGH+ finding exists (this is the default)
linpriv-audit --fail-on high
linpriv-audit --fail-on none       # always exit 0

# Use a config file instead of repeating flags every run
linpriv-audit --config linpriv.yml

# Verbose logging, written to a file
linpriv-audit -vv --log-file scan.log

# Plain text, no ANSI colors (CI logs)
linpriv-audit --no-color

# See everything available
linpriv-audit --list-modules
linpriv-audit --help
linpriv-audit --version
```

### Config file

Copy `linpriv.example.yml` to `./linpriv.yml` (or `~/.config/linpriv/config.yml`
for a global default) to set defaults without repeating CLI flags. CLI flags
always override the config file.

### Reports

Written to `--out-dir` (default `./report_output/`):
- `privesc_report.txt` — plain text
- `privesc_report.html` — styled, color-coded by severity, includes the
  posture score — good for screenshots or attaching to a writeup
- `privesc_report.json` — machine-readable, for CI pipelines or further
  tooling (includes the score, summary counts, and full finding list)

## What it checks

| Module | Key | What it finds |
|---|---|---|
| System Info | `sysinfo` | Current user, groups, kernel, OS release, root status |
| SUID/SGID Binaries | `suid` | SUID/SGID binaries, flagged against a GTFOBins-style dangerous-binary list |
| File/Directory Permissions | `permissions` | World-writable files/dirs, permissions on `/etc/passwd`, `/etc/shadow`, `/etc/sudoers`, `/etc/group` |
| Services & Sudo | `services` | Root-run systemd services with writable `ExecStart` targets; `sudo -l` NOPASSWD / ALL=(ALL) rules |
| Cron Jobs | `cron` | Root cron entries referencing non-root-writable scripts |
| Kernel & OS | `kernel` | Kernel version + OS release, flagged if older than a staleness baseline |
| Capabilities | `capabilities` | File capabilities (`cap_setuid`, `cap_sys_admin`, etc.) — the modern alternative to SUID that most checklists miss |
| PATH Hijacking | `path` | Writable directories in `$PATH`, relative/empty `PATH` entries |
| Privileged Groups | `groups` | Membership in `docker`, `lxd`/`lxc`, or `disk` — each functionally equivalent to root via documented escape techniques |
| NFS / Backdoors / LD_PRELOAD | `misc` | `no_root_squash` NFS exports, extra UID-0 accounts in `/etc/passwd`, writable `/etc/ld.so.preload` |

## Risk scoring

Every scan produces an aggregate **0-100 posture score** with a letter grade
(A-F), weighted by finding severity with diminishing returns per additional
finding of the same severity (so 20 duplicate LOW findings don't swamp the
score the way one real CRITICAL does). Shown in the terminal, and included
in all three report formats.

## Project structure

```
linpriv-toolkit/
├── main.py                      # CLI entry point (rich terminal UI + orchestration)
├── pyproject.toml               # packaging metadata, pytest config
├── requirements.txt
├── linpriv.example.yml          # sample config file
├── Dockerfile / .dockerignore
├── LICENSE
├── CHANGELOG.md
├── modules/
│   ├── common.py                 # Finding dataclass, Severity levels, run_cmd(), logging setup
│   ├── config.py                  # YAML config loading (CLI > config file > defaults)
│   ├── scoring.py                 # risk score / grade computation
│   ├── sysinfo.py                 # system info collection
│   ├── suid_scan.py               # SUID/SGID binary discovery
│   ├── permissions_scan.py        # weak file/dir permissions
│   ├── services_scan.py           # systemd + sudo misconfig
│   ├── cron_scan.py               # cron job vulnerabilities
│   ├── kernel_scan.py             # kernel version / staleness check
│   ├── capabilities_scan.py       # dangerous Linux capabilities
│   ├── path_scan.py               # PATH hijacking
│   ├── group_scan.py              # docker/lxd/disk group membership
│   ├── misc_scan.py               # NFS, backdoor accounts, LD_PRELOAD
│   └── report.py                  # report generation (text + HTML + JSON)
├── tests/                        # pytest suite (26 tests)
└── .github/workflows/ci.yml      # GitHub Actions: test matrix + Docker build
```

## Development

```bash
pip install -e ".[dev]"
pytest tests/ -v --cov=modules
```

CI runs the test suite across Python 3.9–3.12 and builds/smoke-tests the
Docker image on every push and PR (see `.github/workflows/ci.yml`).

## Design notes

- **Detection only**: every command run is read-only (`find`, `ls`,
  `systemctl show`, `sudo -n -l`, `getcap -r`, cron file reads, `uname -a`).
  Nothing is written to the target system, no binaries are invoked to test
  exploitability, no live exploit code exists anywhere in the codebase.
- **Severity model**: `CRITICAL > HIGH > MEDIUM > LOW > INFO`, sorted in all
  output so the highest-risk findings surface first.
- **Extending it**: add `modules/x_scan.py` with a `scan()` function
  returning a list of `Finding` objects (see `modules/common.py`), then
  register it in `MODULE_MAP` in `main.py`.
- **CI-friendly**: `--fail-on`, `--no-color`, and the JSON report are built
  for wiring this into a pipeline that gates on posture score or finding
  severity.

## Legal

This tool is intended for auditing systems you own or are explicitly
authorized to assess. See `LICENSE` for the full license and an additional
authorized-use note.
