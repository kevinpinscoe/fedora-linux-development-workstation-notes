# fedora-linux-development-workstation-notes

Personal notes, observations, and fixes for a **Fedora Linux** developer workstation and desktop running **KDE Plasma 6**.

## System context

| Item | Detail |
|------|--------|
| Host | kevin.kevininscoe.com |
| OS | Fedora 42 (Adams) → 43 |
| Kernel | 6.19.13-100.fc42.x86_64 |
| Desktop | KDE Plasma 6 |
| GPU | AMD Radeon PRO WX 7100 (amdgpu, open-source driver) |
| CPU | AMD Ryzen 5 5500 |
| RAM | 128 GB DDR4 |

## Repository layout

```
fedora-linux-development-workstation-notes/
├── CLAUDE.md                 # Repo conventions + system context (for coding assistants)
├── README.md                  # Repo index (this file)
├── RUNBOOK.md                 # Root runbook index → points to subdirectory runbooks
├── TODO.md                    # Open host tasks (kernel 7.x / GCN4 GPU research)
├── SELINUX_SETUP.md           # SELinux + container labeling notes and fixes
├── fedora-42-to-43-upgrade/   # In-place upgrade notes (Fedora 42 → 43)
│   ├── FEDORA-UPGRADE-43.md              # Full upgrade checklist (pre-upgrade → QA sign-off)
│   ├── PRE-UPGRADE-BASELINE-2026-07-31.md  # Host state captured the night before the upgrade
│   ├── notes-container-inventory.md      # Inventory of the 44 Docker Compose stacks
│   ├── notes-fedora43-upgrade-planning.md  # Research notes and key findings
│   └── notes-system-profile.md           # Hardware, disk layout, and key services
├── kde/
│   └── RUNBOOK.md             # KDE Plasma desktop ops runbook (Plasma/KWin, global shortcuts)
└── garage/
    └── RUNBOOK.md             # Garage (S3-compatible) ops runbook
```

## Contents

### `fedora-42-to-43-upgrade/`

Notes and a step-by-step checklist for the in-place upgrade from Fedora 42 to Fedora 43.

| File | Description |
|------|-------------|
| `FEDORA-UPGRADE-43.md` | Full upgrade checklist: pre-upgrade prep, backups, the upgrade itself, post-upgrade verification (OS, Docker, containers, SELinux, Snap, AMD GPU), and a QA sign-off matrix |
| `notes-fedora43-upgrade-planning.md` | Research notes and key findings from planning the upgrade: breaking changes in F43 (DNF 5, RPM 6, glibc 2.42), Docker/container concerns, and known upgrade failure patterns |
| `notes-system-profile.md` | Hardware inventory, disk layout, and key services on the host |
| `notes-container-inventory.md` | Inventory of the 44 Docker Compose stacks in `/opt/containers`, including which have systemd units, backup timers, and RUNBOOK status |
| `PRE-UPGRADE-BASELINE-2026-07-31.md` | Snapshot of the host taken the night before the F43 upgrade — package versions, pre-existing failed units, container fleet, backup timers, third-party repo readiness. Compare against it during QA so pre-existing faults are not read as upgrade damage. Port and SELinux-confinement detail is held back in an untracked `.local.md` companion, since this repo is public. |

## Runbooks

Operational runbooks are indexed by the root [`RUNBOOK.md`](RUNBOOK.md):

| Runbook | Covers |
|---------|--------|
| [`kde/RUNBOOK.md`](kde/RUNBOOK.md) | KDE Plasma desktop — Plasma/KWin recovery, global shortcuts (incl. the `Meta+N` Obsidian quick-capture into the `~/notes` vault), KRunner, and session plumbing |
| [`garage/RUNBOOK.md`](garage/RUNBOOK.md) | Garage S3-compatible object storage backend |

## Open tasks

Outstanding host work is tracked in [`TODO.md`](TODO.md) — currently researching whether kernel 7.x
still supports the GCN4-era Radeon PRO WX 7100. Container-level work (the Glean decommission) moved
to `/opt/containers/TODO.md` on 2026-08-01 (Glean is now stopped and disabled, with its data
retained pending a decision on the stored articles).

## Key issues documented

- **Docker CE + kernel 6.17** — Kernel 6.17 (shipped with Fedora 43) removes legacy `ip_tables` modules that Docker v28.x requires. Upgrade Docker CE to v29.x *before* upgrading Fedora.
- **AMD amdgpu page faults (kernels 6.17.9–6.18.3)** — Regression affecting some AMD GPU hardware; resolved by 6.18.4+ and 6.19.x. Mitigation: retain multiple kernel entries in GRUB.
- **KDE Plasma duplicate packages** — Plasma 6.5.1 introduced duplicate package conflicts during upgrade; resolved via `dnf distro-sync --skip-broken` or `dnf downgrade krunner`.
- **Wine upgrade conflict** — Remove Wine before running `dnf system-upgrade download`.
- **Woodpecker CI agent restart loop** — Fixed 2026-04-26: pinned to `v3.7.0`, set `WOODPECKER_BACKEND=docker`, added `security_opt: label:disable` for Docker socket SELinux access.

## Environment notes

- SELinux is enforcing (targeted policy).
- 17 Docker Compose services are managed via systemd and backed up with per-container `backup.service`/`backup.timer` units.
- Kasm Workspaces 1.18.1 lives at `/opt/kasm` and is managed separately from the Docker Compose stack.
- System backup uses `rsync` mirrors for `/`, `/boot`, and `/home`; Docker `overlay2` and volumes are excluded and must be snapshotted separately.
