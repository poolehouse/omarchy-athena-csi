# Omarchy × Athena OS × CSI Linux

**A snapshot-first, fully reversible integration of offensive-security and OSINT
tooling into an Arch + Hyprland desktop.**

This project merges the tooling of two security-focused distributions into a
daily-driver [Omarchy](https://omarchy.org/) (Arch + Hyprland) system: **Athena
OS**'s runtime hardening and cyber-toolkit roles, and **CSI Linux**'s OSINT and
case-management workflow — without ever putting boot integrity at risk. Every
phase opens with a snapshot, every touched file is backed up first, and every
change has a documented undo.

## Why this exists

- **Omarchy** is a polished Arch + Hyprland desktop, but it is not a security
  distribution.
- **Athena OS** is Arch-first, so its packages map cleanly onto an Arch base.
- **CSI Linux** is Debian/apt-based, so only its portable Python/Qt tooling and
  its case format cross over.
- The goal is a single machine that is a genuine daily driver *and* a legitimate
  cyber / OSINT workstation — without gambling on boot.

## Components

| Source | Contributes | Lands as |
|---|---|---|
| **Omarchy** (Arch + Hyprland) | Base OS, native snapshot/rollback chain (snapper + Limine) | untouched base |
| **Athena OS** | Runtime hardening (`athena-settings`), cyber-toolkit roles, gateway tooling, repository + keyring | Athena repo (`[athena]`) |
| **BlackArch** | Broad offensive-tool package set | BlackArch repo (`strap.sh`) |
| **CSI Linux** | OSINT tooling + CSI-native case workflow | portable Python/Qt tooling only |

## Design principles

1. **Snapshot before every phase** — root and home.
2. **Back up every touched file** — timestamped copy before any edit.
3. **No bootloader or kernel-cmdline edits** during the tooling phases.
4. **Existing tools win** — an already-installed binary is preferred; collisions
   are logged in a replacement ledger rather than silently overwritten.
5. **Gate between phases** — the prior phase must pass its verification checklist
   and be signed off before the next begins.
6. **Runtime hardening only** — no third-party kernel swap and no LSM
   reconfiguration on a gaming/Steam daily driver.

## Failsafe / rollback model

Tiered, from cheapest to last resort:

| Tier | Mechanism | Scope |
|---|---|---|
| **0** | Boot a snapshot from the boot menu, then "restore now" (replace the root subvolume) | whole system, 1:1 |
| **1** | `snapper undochange <base>..<cur> <paths>` | file-level, no reboot |
| **2** | Per-step reversal (each change names its own undo) | individual changes |
| **3** | Rescue boot + subvolume swap / cached package downgrade | worst case, boot partition untouched |

The full procedure lives in `rollback.md`.

## Phases

| Phase | Scope | Status |
|---|---|---|
| **0** | Safeguards + baseline: snapshot configuration, package/config backups, rollback-chain verification | **done** |
| **1** | Athena + BlackArch repository bootstrap; runtime hardening (`athena-settings`); firewall; sandboxing (Firejail); gateway tooling | **in progress** |
| **2** | Athena cyber-toolkit roles, one section at a time: osint → network → forensic → full catalog | planned |
| **3** | CSI Linux OSINT + case management; partner handoff via a hash-locked evidence bundle | planned |

**Deferred by design** (documented, not executed): third-party kernel swap,
AppArmor LSM activation, USBGuard lockdown, a dedicated SIEM VM.

## Repository bootstrap mechanics

- **Athena:** keyring and mirrorlist are built from upstream PKGBUILDs; the
  Athena repository block is appended **last** so the official and base-distro
  repositories keep precedence. The imported master key fingerprint is verified
  after `pacman-key --populate`.
- **BlackArch:** bootstrapped with the official `strap.sh`. Because upstream
  ships its keyring-signature check disabled, the imported master key
  fingerprint is verified after the run as the compensating control.

## Capability map

Installed incrementally, one role section at a time, with a collision check
against the base system first:

| Area | Representative tooling |
|---|---|
| OSINT | `sherlock`, `theHarvester`, `recon-ng`, `spiderfoot`, `ghunt` |
| Network | `nmap`, `masscan`, `bettercap`, `wireshark` |
| Web | `sqlmap`, `ffuf`, `gobuster`, `burpsuite` |
| Forensics | `sleuthkit`, `autopsy`, `volatility3`, `foremost` |
| General | `metasploit`, `john`, `hashcat`, `hydra`, `aircrack-ng` |

**CSI Linux portability:** the portable pieces are the `csilibs` Python package,
CSI-Manager (`manageapis.py`), the CSI-Utilities one-file tools, OnionSearch,
Recon-Browser, and SpiderFoot. Debian-only components (the apt-based setup and
Powerup wrappers) are intentionally excluded.

**Case workflow:** cases keep the CSI-native `Cases/<CaseID>/` layout with a
`caseinfo.txt`, and hand off through an existing hash-locked evidence-bundle
export (SHA-256 manifest) — no new transport is invented.

## Verification approach

- Backups are **diff-verified** against the live files, not just copied.
- Snapshot identifiers are recorded per phase in an audit runbook.
- Each phase closes with an explicit checklist and a human gate.
- Package changes are captured as before/after package lists for exact reversal.

## Repository layout

| File | Purpose |
|---|---|
| `runbook.md` | Chronological audit log: every command, snapshot, change, and its undo |
| `rollback.md` | Tiered restore / undo procedures |
| `conflicts.md` | Package and config collision ledger (replacement map) |
| `backups/` | Timestamped copies of package lists, configs, and touched files |
| `state/` | Machine-readable markers: current phase, last good snapshot, next actions |

## Scope & ethics

- Authorized personal-lab integration; offensive tooling is installed for
  sanctioned testing, CTFs, and coursework.
- Third-party tooling keeps its upstream license and attribution; unlicensed
  tools are not redistributed.
- No boot-affecting changes are made without a verified rollback path.

## AI transparency

This work is built with AI assistance for planning, drafting, and executing
routine steps. All privileged and system-level changes are human-reviewed and
human-executed. This document is sanitized by construction: it contains no
personal identifiers, credentials, hostnames, addresses, or private
infrastructure details.
