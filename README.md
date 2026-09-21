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
| **Athena OS** | Runtime hardening (`athena-settings`), cyber-toolkit roles, gateway tooling, repository + keyring | `[athena]` repo — **installed & active** |
| **BlackArch** | Broad offensive-tool package set | `[blackarch]` repo — **installed & active** |
| **CSI Linux** | OSINT tooling + CSI-native case workflow | portable Python/Qt tooling only |

## Design principles

1. **Snapshot before every phase** — root and home.
2. **Back up every touched file** — timestamped copy before any edit.
3. **No bootloader or kernel-cmdline edits** during the tooling phases.
4. **Existing tools win** — an already-installed binary is preferred; collisions
   are logged in a replacement ledger rather than silently overwritten.
5. **Additive role installs** — cyber-toolkit roles install with `pacman -S
   --needed` (accumulate), so switching roles never uninstalls prior tooling.
6. **Gate between phases** — the prior phase must pass its verification checklist
   (`pacman -Qkk` clean, no enabled services, firewall ruleset unchanged) and be
   signed off before the next begins.
7. **Runtime hardening only** — no third-party kernel swap and no LSM
   reconfiguration on a gaming/Steam daily driver.

## Failsafe / rollback model

Tiered, from cheapest to last resort:

| Tier | Mechanism | Scope |
|---|---|---|
| **0** | Boot a snapshot from the boot menu, then "restore now" (replace the root subvolume) | whole system, 1:1 |
| **1** | `snapper undochange <base>..<cur> <paths>` | file-level, no reboot |
| **2** | Per-step reversal (each change names its own undo) | individual changes |
| **3** | Rescue boot + subvolume swap / cached package downgrade | worst case, boot partition untouched |

Tier 2-3 procedures are documented per change during execution in a private
runbook kept alongside the working copy.

## Phases

| Phase | Scope | Status |
|---|---|---|
| **0** | Safeguards + baseline: snapshot configuration, package/config backups, rollback-chain verification | **done** |
| **1** | Athena + BlackArch repository bootstrap; runtime hardening (`athena-settings`); firewall audit; gateway tooling | **done** |
| **2** | Athena cyber-toolkit roles, one section at a time: osint → network → forensic → catalog | osint, network, forensic **done**; remaining catalog **deferred** |
| **3** | CSI Linux OSINT + case management; partner handoff via a hash-locked evidence bundle | planned |

**Deferred by design** (documented, not executed): third-party kernel swap,
AppArmor LSM activation, USBGuard lockdown, a dedicated SIEM VM.
**Firejail is intentionally not installed** — sandboxing a Steam/gaming daily
driver carries a real breakage risk, so it is held rather than wrapped around
the base system.

## Implemented configuration (as exercised)

- **Repositories:** Athena keyring `athena-keyring` + mirrorlist
  `athena-mirrorlist` built from upstream PKGBUILDs (the Athena repo block is
  appended **last** so official/base-distro repos keep precedence); BlackArch
  bootstrapped via official `strap.sh`. Result: **92 Athena packages** and
  **5050 BlackArch packages** resolvable against an untouched
  core/extra/multilib/omarchy base.
- **Runtime hardening (`athena-settings`):** installed and scoped to runtime
  knobs (`kernel.kptr_restrict=1`, etc.). A passwordless `sudo` drop-in shipped
  by the package was **removed** to preserve the base OS's
  password-everywhere posture — the base configuration wins on conflicts.
- **Firewall (UFW):** already active on the base image; audited and left
  unchanged by every phase (ruleset diffed against baseline after each step).
- **Gateway tooling:** `nist-feed`, `mitre-attack-navigator`, `athena-nexus`
  (with their notification/schedule deps) installed and verified clean.
- **Cyber-toolkit roles:** installed additively and verified. osint (26 pkgs),
  network (59 pkgs), forensic (61 pkgs) — every package reports **0 altered
  files** against the archive; no systemd unit is left enabled; the network
  role's driver packages triggered a normal initramfs rebuild (harmless);
  the forensic mapping `exiftool` → `perl-image-exiftool` needed no custom
  handling.
- **Snapshots:** every phase and role install is bracketed by root + home
  snapper baselines (`pre`/`post`), enabling Tier 0-1 rollback at any point.

## Repository bootstrap mechanics

- **Athena:** keyring and mirrorlist are built from upstream PKGBUILDs; the
  repository block is appended **last** so the official and base-distro
  repositories keep precedence. The imported master key fingerprint is verified
  after `pacman-key --populate`.
- **BlackArch:** bootstrapped with the official `strap.sh`. Because upstream
  ships its keyring-signature check disabled, the imported master key
  fingerprint is verified after the run as the compensating control.

## Capability map

Installed additively, one role section at a time, with a collision check
against the base system first. Cell shading reflects execution status.

| Area | Representative tooling | Status |
|---|---|---|
| OSINT | `sherlock`, `theHarvester`, `recon-ng`, `spiderfoot`, `ghunt` | installed & verified |
| Network | `nmap`, `masscan`, `bettercap`, `wireshark`, `zeek` | installed & verified |
| Forensics | `sleuthkit`, `volatility3`, `foremost`, `bulk_extractor`, `regripper` | installed & verified |
| Web | `sqlmap`, `ffuf`, `gobuster`, `burpsuite` | pending (catalog deferred) |
| General | `metasploit`, `john`, `hashcat`, `hydra`, `aircrack-ng` | pending (catalog deferred) |

**CSI Linux portability:** the portable pieces are the `csilibs` Python package,
CSI-Manager (`manageapis.py`), the CSI-Utilities one-file tools, OnionSearch,
Recon-Browser, and SpiderFoot. Debian-only components (the apt-based setup and
Powerup wrappers) are intentionally excluded.

**Case workflow:** cases keep the CSI-native `Cases/<CaseID>/` layout with a
`caseinfo.txt`, and hand off through an existing hash-locked evidence-bundle
export (SHA-256 manifest) — no new transport is invented.

## Verification approach

- Backups are **diff-verified** against the live files, not just copied.
- Snapshot identifiers are recorded per phase in the audit runbook.
- Each phase closes with an explicit checklist and a human gate:
  `pacman -Qkk` clean (0 altered files), **no enabled systemd units**, and a
  firewall ruleset diff against the pre-phase snapshot.
- Package changes are captured as before/after package lists for exact reversal.

## Repository layout

The working copy contains the execution artifacts (private): the audit runbook,
rollback procedures, collision ledger, timestamped backups, and a
machine-readable status file. This public repository publishes this sanitized
README and, over time, the reusable phase scripts.

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