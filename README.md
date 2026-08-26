# linpriv-audit

**Detection-only Linux privilege escalation audit toolkit.**

Scans a Linux host for common privilege escalation vectors, scores overall
security posture (0–100, A–F), and produces a report in text, HTML, and
JSON. Every check is read-only enumeration — the tool never executes
exploit code or modifies the target system.

![Python](https://img.shields.io/badge/python-3.8%2B-blue)
![Platform](https://img.shields.io/badge/platform-Linux-lightgrey)
![License](https://img.shields.io/badge/license-MIT-green)

```
 _     _       ____       _
| |   (_)_ __ |  _ \ _ __(_)_   __
| |   | | '_ \| |_) | '__| \ \ / /
| |___| | | | |  __/| |  | |\ V /
|_____|_|_| |_|_|   |_|  |_| \_/
```

## Contents

- [What it does](#what-it-does)
- [What it checks](#what-it-checks)
- [Install](#install)
- [Usage](#usage)
- [Config file](#config-file)
- [Reports](#reports)
- [Risk scoring](#risk-scoring)
- [Project structure](#project-structure)
- [Development](#development)
- [Design notes](#design-notes)
- [Legal / responsible use](#legal--responsible-use)

## What it does

Privilege escalation happens when a low-privileged user gains root access
through weak configurations, mismanaged permissions, or vulnerable
services. This toolkit automates the enumeration side of that audit —
the same checks a penetration tester runs by hand, or a blue-team defender
runs to catch drift before an attacker finds it.

It's built around a simple contract: every check module returns a list of
`Finding` objects (title, severity, exploitability note, mitigation,
evidence), which get aggregated into a severity-ranked report and an
overall posture score.

**Nothing here executes an exploit.** It reads config, checks permissions,
and reports — it does not overwrite files, spawn shells, or touch anything
on the target system.

## What it checks

| Module | Key | What it finds |
|---|---|---|
| System Info | `sysinfo` | Current user, groups, kernel, OS release, root status |
| SUID/SGID Binaries | `suid` | SUID/SGID binaries, flagged against a GTFOBins-style dangerous-binary list |
| File/Directory Permissions | `permissions` | World-writable files/dirs; permissions on `/etc/passwd`, `/etc/shadow`, `/etc/sudoers`, `/etc/group` |
| Services & Sudo | `services` | Root-run systemd services with writable `ExecStart` targets; `sudo -l` NOPASSWD / `ALL=(ALL)` rules |
| Cron Jobs | `cron` | Root cron entries referencing scripts writable by non-root users |
| Kernel & OS | `kernel` | Kernel version + OS release, flagged if older than a staleness baseline |
| Capabilities | `capabilities` | File capabilities (`cap_setuid`, `cap_sys_admin`, etc.) — the modern alternative to SUID that most checklists miss |
| PATH Hijacking | `path` | Writable directories in `$PATH`, relative/empty `PATH` entries |
| Privileged Groups | `groups` | Membership in `docker`, `lxd`/`lxc`, or `disk` — each functionally equivalent to root via documented escape techniques |
| NFS / Backdoors / LD_PRELOAD | `misc` | `no_root_squash` NFS exports, extra UID-0 accounts in `/etc/passwd`, writable `/etc/ld.so.preload` |

## Install

Requires **Linux** and **Python 3.8+**. This tool reads `/etc/shadow`,
`systemctl`, `sudo -l`, cron files, and other Linux-specific interfaces, so
it only runs on Linux — running it elsewhere fails with a clear message
rather than a crash.

**pip (recommended):**
```bash
git clone https://github.com/<your-username>/linpriv-audit.git
cd linpriv-audit
pip install -e .
linpriv-audit --version
```

**Without installing, just the dependencies:**
```bash
pip install -r requirements.txt
python3 main.py --version
```

**Docker:**
```bash
docker build -t linpriv-audit .
docker run --rm -v "$PWD/report_output:/app/report_output" linpriv-audit
```
> A container is namespace-isolated from the host, so a scan run this way
> audits the *container's* filesystem/config, not the host machine. Use it
> for testing the tool itself, not for auditing a real host.

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

# CI-friendly: exit non-zero if a HIGH+ finding exists (default behavior)
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
```

## Config file

Copy `linpriv.example.yml` to `./linpriv.yml` (project-local) or
`~/.config/linpriv/config.yml` (global default) to set options without
repeating CLI flags every run. CLI flags always take precedence.

```yaml
modules: all
out_dir: ./report_output
exclude_paths:
  - /proc
  - /sys
  - /dev
  - /tmp
fail_on: high
show_info: false
```

## Reports

Written to `--out-dir` (default `./report_output/`):

- **`privesc_report.txt`** — plain text
- **`privesc_report.html`** — styled, color-coded by severity, includes the
  posture score — good for screenshots or attaching to a writeup
- **`privesc_report.json`** — machine-readable, for CI pipelines or further
  tooling (includes the score, summary counts, and full finding list)

## Risk scoring

Every scan produces an aggregate **0–100 posture score** with a letter
grade (A–F), weighted by finding severity with diminishing returns per
additional finding of the same severity — so 20 duplicate LOW findings
don't swamp the score the way one real CRITICAL does. Shown in the
terminal and included in all three report formats.

## Project structure

```
linpriv-audit/
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
├── tests/                        # pytest suite
└── .github/workflows/ci.yml      # GitHub Actions: test matrix + Docker build
```

## Development

```bash
pip install -e ".[dev]"
pytest tests/ -v --cov=modules
```

CI runs the test suite across Python 3.9–3.12 and builds/smoke-tests the
Docker image on every push and PR — see `.github/workflows/ci.yml`.

To add a new check: create `modules/x_scan.py` with a `scan()` function
returning a list of `Finding` objects (see `modules/common.py`), then
register it in `MODULE_MAP` in `main.py`.

## Design notes

- **Detection only** — every command run is read-only (`find`, `ls`,
  `systemctl show`, `sudo -n -l`, `getcap -r`, cron file reads, `uname -a`).
  Nothing is written to the target system, no binaries are invoked to test
  exploitability, and no exploit code exists anywhere in the codebase.
- **Severity model** — `CRITICAL > HIGH > MEDIUM > LOW > INFO`, sorted in
  all output so the highest-risk findings surface first.
- **CI-friendly** — `--fail-on`, `--no-color`, and the JSON report are
  built for wiring this into a pipeline that gates on posture score or
  finding severity.

## Legal / responsible use

This tool is intended for auditing systems you own or are explicitly
authorized to assess. Running it against systems without authorization may
violate computer misuse laws in your jurisdiction (e.g. the CFAA in the
US, the Computer Misuse Act in the UK). You are responsible for ensuring
you have appropriate authorization before use. See `LICENSE` for the full
license terms.

## License

MIT — see [LICENSE](LICENSE).
