# Linux Privilege Escalation Automation Toolkit

A **detection-only** auditing tool that scans a Linux system for privilege
escalation misconfigurations and produces a severity-ranked report — with
a polished, color-coded terminal UI and text/HTML/JSON report output.
It never runs exploit code: every check is read-only enumeration.

## Install

```bash
pip install -r requirements.txt
```

Requires Python 3.8+ and the `rich` library for terminal output.

## Usage

```bash
# Run every module with the full colored terminal UI
python3 main.py

# Run specific modules only
python3 main.py modules suid,permissions,cron

# Include INFO-level findings in the terminal output (full inventory)
python3 main.py show-info

# Custom output directory
python3 main.py out-dir /tmp/audit_results

# Plain text, no ANSI colors (e.g. for CI logs)
python3 main.py no-color

# List every available module and exit
python3 main.py list-modules
```

Outputs (written to `out-dir`, default `./report_output`):
- `privesc_report.txt` — plain-text report
- `privesc_report.html` — styled HTML report (good for screenshots/presentations)
- `privesc_report.json` — machine-readable report (CI pipelines, SIEM ingestion, further tooling)

## What it checks

| Module | Key | What it finds |
|---|---|---|
| System Info | `sysinfo` | Current user, groups, kernel, OS release, root status |
| SUID/SGID Binaries | `suid` | SUID/SGID binaries, flagged against a GTFOBins-style dangerous-binary list |
| File/Directory Permissions | `permissions` | World-writable files/dirs, permissions on `/etc/passwd`, `/etc/shadow`, `/etc/sudoers`, `/etc/group` |
| Services & Sudo | `services` | Root-run systemd services with writable `ExecStart` targets; `sudo -l` NOPASSWD / ALL=(ALL) rules |
| Cron Jobs | `cron` | Root cron entries referencing non-root-writable scripts |
| Kernel & OS | `kernel` | Kernel version + OS release, flagged if older than a staleness baseline |
| **Capabilities** | `capabilities` | File capabilities (`cap_setuid`, `cap_sys_admin`, etc.) — the modern alternative to SUID that most checklists miss |
| **PATH Hijacking** | `path` | Writable directories in `$PATH`, relative/empty `PATH` entries |
| **Privileged Groups** | `groups` | Membership in `docker`, `lxd`/`lxc`, or `disk` — each functionally equivalent to root via documented escape techniques |
| **NFS / Backdoors / LD_PRELOAD** | `misc` | `no_root_squash` NFS exports, extra UID-0 accounts in `/etc/passwd`, writable `/etc/ld.so.preload` |

The last four modules go beyond a typical checklist-style tool and cover
attack surface that's common in real environments but frequently missed.

## Project structure

```
linpriv-toolkit/
├── main.py                      # CLI entry point: rich-powered terminal UI + orchestration
├── requirements.txt
├── modules/
│   ├── common.py                # Finding dataclass, Severity levels, run_cmd() helper
│   ├── sysinfo.py                # system info collection
│   ├── suid_scan.py              # SUID/SGID binary discovery
│   ├── permissions_scan.py       # weak file/dir permissions
│   ├── services_scan.py          # systemd + sudo misconfig
│   ├── cron_scan.py              # cron job vulnerabilities
│   ├── kernel_scan.py            # kernel version / staleness check
│   ├── capabilities_scan.py      # dangerous Linux capabilities
│   ├── path_scan.py              # PATH hijacking
│   ├── group_scan.py             # docker/lxd/disk group membership
│   ├── misc_scan.py              # NFS, backdoor accounts, LD_PRELOAD
│   └── report.py                 # report generation (text + HTML + JSON)
└── README.md
```

