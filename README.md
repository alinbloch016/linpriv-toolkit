# Linux Privilege Escalation Automation Toolkit

A **detection-only** toolkit that scans a Linux system to detect privilege
escalation opportunities. It scans, analyzes, and reports potential risks
without executing harmful exploitation steps.

## Install

```bash
pip install -r requirements.txt
```

Requires Python 3.8+ and Linux (the checks read `/etc/shadow`, `systemctl`,
`sudo -l`, cron files, etc., which don't exist on other platforms).

## Usage

```bash
# Run all five checks
python main.py

# Run specific modules only
python main.py --modules suid,permissions,cron

# Include INFO-level findings in the terminal output
python main.py --show-info

# Custom output directory
python main.py --out-dir /tmp/audit_results

# Plain text, no ANSI colors
python main.py --no-color

# List available modules
python main.py --list-modules
```

The report is written to `out-dir` (default `./report_output/`) as
`privesc_report.txt`.

## Workflow

Following the project's four-step design:

1. **System Information Collection** - current user, groups, kernel
   version, OS release, whether the user is root or limited.
2. **Privilege Escalation Vector Scanning** - runs the five modules below.
3. **Analysis Engine** - ranks findings by severity.
4. **Report Generation** - exports a structured report with, for every
   finding: the finding itself, its severity, its exploitation
   possibility, and suggested mitigation steps.

## What it checks

| Module | Key | What it finds |
|---|---|---|
| SUID/SGID Binary Discovery | `suid` | SUID/SGID binaries across the filesystem, cross-referenced against a GTFOBins-style list of binaries with known exploit paths (`awk`, `find`, `perl`, `vim`, etc.) |
| Weak File & Directory Permissions | `permissions` | World-writable files/directories; permissions on `/etc/passwd`, `/etc/shadow`, `/etc/sudoers`, `/etc/group` |
| Misconfigured Services | `services` | systemd services running as root with a writable `ExecStart` target; `sudo -l` NOPASSWD / `ALL=(ALL)` rules |
| Cron Job Vulnerabilities | `cron` | Root-run cron entries (`/etc/crontab`, `/etc/cron.d`, `cron.{hourly,daily,weekly,monthly}`) that reference scripts writable by non-root users |
| Kernel Exploit Detection | `kernel` | Kernel version and OS release, flagged if older than a staleness baseline (manual CVE cross-reference required - no live exploits are attempted) |

## Project structure

```
linpriv-toolkit/
├── main.py                      # orchestrator: the 4-step workflow
├── requirements.txt
├── modules/
│   ├── common.py                 # Finding dataclass, Severity levels, run_cmd()
│   ├── sysinfo.py                 # STEP 1: system info collection
│   ├── suid_scan.py               # SUID/SGID Binary Discovery
│   ├── permissions_scan.py        # Weak File & Directory Permissions
│   ├── services_scan.py           # Misconfigured Services (+ sudo -l)
│   ├── cron_scan.py               # Cron Job Vulnerabilities
│   ├── kernel_scan.py             # Kernel Exploit Detection
│   └── report.py                  # STEP 4: report generation
└── README.md
```

## Tools & technologies

- **Python** - implementation language
- **rich** - terminal output (colors, progress bars, panels)
- Linux utilities invoked (read-only): `find`, `systemctl show`, `sudo -n -l`,
  `crontab` / `/etc/cron.*`, `uname -a`

## Design notes

- **Detection only**: every command run is read-only (`find`, `systemctl
  show`, `sudo -n -l`, cron file reads, `uname -a`). Nothing is written to
  the target system, no binaries are invoked to test exploitability, and no
  exploit code exists anywhere in the codebase.
- **Severity model**: `CRITICAL > HIGH > MEDIUM > LOW > INFO`.

## Legal / responsible use

This tool is intended for auditing systems you own or are explicitly
authorized to assess. Running it against systems without authorization may
violate computer misuse laws in your jurisdiction.
