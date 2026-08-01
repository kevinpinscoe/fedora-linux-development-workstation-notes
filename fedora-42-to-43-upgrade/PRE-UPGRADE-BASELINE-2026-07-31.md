# Pre-Upgrade Baseline — captured 2026-07-31 18:00 EDT

Snapshot of the FLDW immediately before the Fedora 42 → 43 upgrade. Its purpose is to give the
post-upgrade QA pass something real to compare against, so that pre-existing faults are not
mistaken for upgrade damage.

**Compare this file against the live host during QA. Anything already broken here is not the
upgrade's fault.**

> ### Deliberate changes made AFTER this snapshot, on 2026-08-01
>
> Two things below are intentionally no longer true. Do not treat either as upgrade damage:
>
> - **Salt was removed entirely** — `salt`, `salt-master`, `salt-minion` and all of `/etc/salt`,
>   `/var/cache/salt`, `/var/log/salt`. Ports 4505/4506 no longer listen. It was outmoded and was
>   also about to break on Python 3.14.
> - **Glean was stopped and disabled** — its six containers are down and `glean.service` is
>   disabled, so the running-container count is **67, not 73**. Nothing was destroyed: the unit
>   file and all eight volumes are retained pending a decision on its stored articles.
>   See `/opt/containers/TODO.md`.

> **Redacted for publication.** This repository is public, so the port map and the list of stacks
> running with SELinux confinement disabled are **not** in this copy. They live beside it in
> `PRE-UPGRADE-BASELINE-2026-07-31.local.md`, which is gitignored and stays on the host. Use the
> `.local.md` copy when actually running QA.

---

## OS and packages

| Item | Value at baseline |
|---|---|
| Release | Fedora 42 (Adams) — **EOL, moved to `archives.fedoraproject.org`** |
| Running kernel | `6.19.14-108.fc42.x86_64` |
| Installed kernels | `6.19.8-100`, `6.19.13-100`, `6.19.14-108` (all fc42) |
| RPM | 4.20.1 |
| DNF | dnf5 5.2.18.0 (already DNF 5 on F42) |
| SELinux | Enforcing, `selinux-policy-42.24-1.fc42` |
| Plasma | `plasma-desktop-6.6.4-1.fc42`, `plasmashell 6.6.4` |
| Docker engine | **`moby-engine-29.4.2-1.fc42`** (Fedora package — *not* Docker CE) |
| Docker CLI | `docker-cli-29.4.2-1.fc42` |
| Compose | `docker-compose-5.1.2-1.fc42` |
| containerd | `containerd.io-2.2.4-1.fc42` (from the Docker repo) |
| httpd | 2.4.66-1.fc42 |
| Tailscale | 1.98.4 |
| snapd | 2.75.2 (snap `snapd` rev is 2.76.1) |
| Salt | `salt` / `salt-master` / `salt-minion` all 3007.5-2.fc42 |

## Core services at baseline

`httpd`, `tailscaled`, `snapd`, `docker`, `certbot-renew.timer` — all **active**. `getenforce` → **Enforcing**.

---

## Pre-existing failed units — NOT caused by the upgrade

`systemctl --failed` already reports these. QA-1 expects zero failed units; that expectation is
wrong on this host. Treat only *new* names as upgrade damage.

| Unit | Note |
|---|---|
| `check-ai-skill-jobs.service` | Silent-fail check for the `~/ai/fedora` background skill jobs |
| `check-changelog-roll.service` | Health/deadman check for the FLDW CHANGELOG month roll |
| `dnsmasq.service` | Failed at baseline |
| `github-poll-for-activity.service` | Polls GitHub for stars/forks |
| `drkonqi-coredump-processor@*.service` (10 instances) | KDE coredump handler noise — routinely present |
| `drill-execstartpost-*.service` | Transient `systemd-run` test unit (`/bin/false`) — a monitoring drill, not a fault |

**Revised 2026-08-01 11:45** — the list drifted during the day:
- `check-pcm-nightly-ingest` was **fixed** and no longer fails
- `check-changelog-roll` and the `drill-execstartpost-*` transient appeared
- `gitea-backup` failed at 03:15 and was **fixed at 11:42** (SELinux label; see the checklist's QA-8)

## Kasm Workspaces — already down before the upgrade

All 8 Kasm containers are **exited** (`kasm_proxy`, `kasm_rdp_https_gateway`, `kasm_rdp_gateway`,
`kasm_agent`, `kasm_manager`, `kasm_api`, `kasm_guac`, `kasm_db`), and `kasm.service` is
**disabled and inactive**. Kasm 1.18.1 is still installed at `/opt/kasm`.

Kasm being down after the upgrade is therefore the *expected* state, not a regression.

---

## Container fleet at baseline

**73 containers running**, from **44 Compose stacks** under `/opt/containers/`
(plus `youtrack.corrupt-20260729`, a quarantined copy, and the non-Compose Kasm stack).

Stacks with an enabled systemd unit: `actualbudget`, `convertx`, `excalidraw`, `garage`,
`gitea-act-runner`, `glean`, `ingest`, `karakeep`, `kroki`, `metabase`, `n8n`, `openbao`,
`pastebooks`, `pkm`, `reader`, `rsshub`, `wallabag`, `watchtower`, `wikijs`, `woodpecker-ci`,
`youtrack`.

Stacks running without a systemd unit (started by Compose directly — these will **not** come back
on their own after a reboot): `argus`, `beszel`, `c3x`, `cadvisor`, `checkmk`, `dashboard`,
`erugo`, `filebrowser`, `gatus`, `gitea`, `glance`, `home_file_server`, `kavita`, `lxconsole`,
`pgadmin`, `picoshare`, `portainer`, `qui`, `rss`, `tool-shed`.

> `gitea` has a backup timer but no service unit — worth confirming how it is meant to start.

### Stacks using `security_opt: label:disable`

**11 stacks** bypass SELinux confinement and must be revalidated after the policy update — up from
the 4 the old checklist listed. **Names redacted from the published copy;** see the `.local.md`
file, or regenerate:

```bash
sudo grep -rl 'label:disable' /opt/containers/*/docker-compose.y*ml \
  | sed 's|/opt/containers/||;s|/docker-compose.*||'
```

### Published ports at baseline — REDACTED

40 ports were published by the fleet at baseline: mostly `127.0.0.1`-bound, with 10 bound to
`0.0.0.0`. **The port-to-service map is redacted from the published copy** — see the `.local.md`
file, or regenerate:

```bash
docker ps --format '{{.Names}}\t{{.Ports}}' | grep -E '0\.0\.0\.0|127\.0\.0\.1' | sort
```

---

## Backup timers at baseline — 38 timers

All were scheduled with a valid next-trigger time and none were in a failed state.

`actualbudget-backup`, `argus-backup`, `backup-beszel`, `backup-core`, `backup-donetick-from-web1-to-local`,
`backup-filebrowser`, `backup-from-web1-to-local`, `backup-matomo-from-mail1-to-local`, `backup-picoshare`,
`backup-qui`, `backup-unclutter-from-web1-to-local`, `backup-web1-openbao`, `backupmysql`, `c3x-backup`,
`checkmk-backup`, `convertx-backup`, `dashboard-backup`, `erugo-backup`, `garage-backup`, `gitea-backup`,
`gitea-backup-verify`, `karakeep-backup`, `kavita-backup`, `kroki-backup`, `lxconsole-backup`,
`metabase-backup`, `n8n-backup`, `openbao-backup`, `pastebooks-backup`, `pgadmin-backup`,
`portainer-backup`, `reader-backup`, `rsshub-backup`, `wallabag-backup`, `wikijs-backup`,
`woodpecker-ci-backup`, `youtrack-backup`, `youtrack-backup-verify`

**Not a timer:** Glean's backup is a cron job — `/etc/cron.d/glean-backup`, 02:30 daily. Cron has no
catch-up, so it silently skips whenever the host is down at 02:30 (as it did on 2026-07-27).

---

## Third-party repository readiness for F43

Verified 2026-07-31 — all resolve for `releasever=43`:

| Repo | F43 status |
|---|---|
| RPM Fusion (free + nonfree) | Available |
| Docker CE repo (`containerd.io`) | Available (f43 and f44 dirs both published) |
| Fedora `moby-engine` | **29.6.2-1.fc43** — stays on 29.x, no downgrade |
| COPR `dejan/lazygit` | Available |
| COPR `ilyaz/LACT` | Available |
| COPR `jdxcode/mise` | Available |
| COPR `lihaohong/yazi` | Available |
| COPR `phracek/PyCharm` | Available |
| Tailscale | Release-agnostic (`$basearch` only, no `$releasever`) |
| 1Password, VS Code, Cursor, LibreWolf, Grafana, kevinpinscoe | Release-agnostic fixed paths |

## Snaps at baseline

`bare`, `core`, `core20`, `core22`, `core24`, `ffmpeg-2404`, `gnome-42-2204`, `gnome-46-2404`,
`gtk-common-themes`, `mesa-2404`, `obs-studio`, `shortwave`, `snapd`
