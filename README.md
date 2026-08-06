# fedora-notes

Personal notes, observations, and fixes for a **Fedora Linux** developer workstation and desktop running **KDE Plasma 6**.

## System context

| Item | Detail |
|------|--------|
| Host | FLDW (Fedora Linux Development Workstation) |
| OS | Fedora 43 (KDE Plasma Desktop Edition) — notes span 42 → 43 |
| Kernel | 7.1.5-101.fc43.x86_64 |
| Desktop | KDE Plasma 6 |
| GPU | AMD Radeon PRO WX 7100 (amdgpu, open-source driver) |
| CPU | AMD Ryzen 5 5500 |
| RAM | 128 GB DDR4 |

## Repository layout

```
fedora-notes/
├── CLAUDE.md                 # Repo conventions + provenance (for coding assistants)
├── README.md                  # Repo index (this file)
├── RUNBOOK.md                 # Root runbook index → points to subdirectory runbooks
├── SELINUX_SETUP.md           # SELinux + container labeling notes and fixes
├── fedora-42-to-43-upgrade/   # In-place upgrade notes (Fedora 42 → 43)
│   ├── FEDORA-UPGRADE-43.md              # Full upgrade checklist (pre-upgrade → QA sign-off)
│   ├── PRE-UPGRADE-BASELINE-2026-07-31.md  # Host state captured the night before the upgrade
│   ├── notes-fedora43-upgrade-planning.md  # Research notes and key findings
│   └── notes-system-profile.md           # Hardware, disk layout, and key services
├── fedora-43-to-44-upgrade/   # In-place upgrade notes (Fedora 43 → 44) — planning only
│   ├── notes-fedora44-upgrade-planning.md  # Research notes; checklist not written yet
│   └── notes-mariadb-11-migration.md       # MariaDB 10.11 → 11.8 migration research
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
| `PRE-UPGRADE-BASELINE-2026-07-31.md` | Snapshot of the host taken the night before the F43 upgrade — package versions, pre-existing failed units, container fleet, backup timers, third-party repo readiness. Compare against it during QA so pre-existing faults are not read as upgrade damage. Port and SELinux-confinement detail is held back in an untracked `.local.md` companion, since this repo is public. |

### `fedora-43-to-44-upgrade/`

**Planning only — nothing scheduled, and the F43 QA pass is not finished.**

| File | Description |
|------|-------------|
| `notes-fedora44-upgrade-planning.md` | Research notes: package diff F43 → F44 read off live metadata, the MariaDB 10.11 → 11.8 migration as the standout risk, third-party repo readiness, prerequisites inherited from F43, and the open F44-vs-skip-to-F45 question |
| `notes-mariadb-11-migration.md` | MariaDB 10.11 → 11.8 research: what is actually on the host (two MediaWiki DBs, ~360 MB, two regenerable MyISAM tables), the supported in-place upgrade path, why the rollback is restore-from-dump, and the `innodb_snapshot_isolation` default change as the most likely real fault |

The actionable checklist (`FEDORA-UPGRADE-44.md`) is deliberately not written yet — see the notes
file's "Next artifact" section for what has to be answered first.

**Package manifests (untracked).** Two `rpm -qa` captures taken 2026-08-02 live in this directory
as `installed-packages-before-fedora-44-2026-08-02.local.txt` (full NEVRA, 8,480 lines) and
`installed-package-names-before-fedora-44-2026-08-02.local.txt` (names only, 8,236 lines). They
exist because the F43 baseline never captured a package list, which made "was this package
installed before the upgrade?" unanswerable during F43 QA. Both carry the `.local.txt` suffix and
are gitignored — **this repo is public**, and an exact software-and-version inventory of an
internet-reachable host is the strongest reconnaissance artifact it could hold. See the planning
notes' "Baseline gap to close before F44" section for how to diff them after an upgrade.

## Runbooks

Operational runbooks are indexed by the root [`RUNBOOK.md`](RUNBOOK.md):

| Runbook | Covers |
|---------|--------|
| [`kde/RUNBOOK.md`](kde/RUNBOOK.md) | KDE Plasma desktop — Plasma/KWin recovery, global shortcuts (incl. the `Meta+N` Obsidian quick-capture into the `~/notes` vault), KRunner, and session plumbing |
| [`garage/RUNBOOK.md`](garage/RUNBOOK.md) | Garage S3-compatible object storage backend |

## Key issues documented

- **Kernel 7.1 on GCN4 / Polaris is fine** — the 6.19 → 7.1 jump in a single step was the largest unknown going into F43 on a 2016 Radeon PRO WX 7100, and no reports either way could be found beforehand. Two instrumented soaks settled it: 35 minutes on the gfx ring (2026-08-02) and 15 minutes driving the UVD and VCE video engines (2026-08-03), both finding **zero** faults, ring timeouts, GPU resets, or DRM errors. Polaris/GCN4 remains fully supported by `amdgpu` and Mesa, not moved to legacy. The card throttled on its 95 W power cap rather than thermally, which is correct behavior. The one real GCN4 limitation is **ROCm**, which dropped `gfx803` years ago — unrelated to graphics or video.
- **Stock Fedora ships no H.264/HEVC VAAPI** — hardware video decode and encode appear broken on *any* GPU, current models included, because Fedora strips those codecs from `mesa-va-drivers` for patent reasons. The symptom is `ffmpeg ... h264_vaapi` failing with `No usable encoding profile found`. Installing `mesa-va-drivers-freeworld` from RPM Fusion took VAAPI profiles from **3 → 15** and restored both decode and encode. Install **both** arches — the `i686` build matters for Steam, Wine, and Bottles, which use the 32-bit graphics stack. This is a packaging gap, not a hardware or driver fault.
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
