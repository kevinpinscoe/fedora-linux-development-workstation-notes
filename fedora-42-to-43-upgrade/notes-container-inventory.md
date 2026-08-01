---
name: Container Inventory
description: All Docker containers in /opt/containers — images, ports, RUNBOOK status, backup timers
type: project
originSessionId: ede1e2fc-66bc-4f7d-96c3-643bbfd91d37
---

> **Regenerated 2026-07-31** from the live host. The April 2026 version of this file described 17
> stacks; the fleet has since roughly tripled. Point-in-time detail (images, ports, health) lives in
> `PRE-UPGRADE-BASELINE-2026-07-31.md` — this file is the structural inventory.

All containers are Docker Compose-based and live in `/opt/containers/`. **Only about half are
managed by a systemd service** — the rest were started with `docker compose up -d` directly and do
not survive a reboot on their own.

**Totals at 2026-07-31:** 44 Compose stacks, 73 running containers.

> **Update 2026-08-01:** Glean was stopped and disabled before the Fedora 43 upgrade — its six
> containers are down and its unit is disabled, leaving **67 running**. Nothing was destroyed: the
> unit file, all eight volumes, and `/opt/containers/glean/` are retained. See `/opt/containers/TODO.md`.

Kasm Workspaces 1.18.1 is at `/opt/kasm` — NOT in `/opt/containers`, and **currently down and
disabled** (all 8 containers exited, `kasm.service` disabled).

## Stacks with an enabled systemd unit

These start at boot. 21 stacks.

| Stack | RUNBOOK | Backup timer |
|---|---|---|
| actualbudget | yes | actualbudget-backup |
| convertx | yes | convertx-backup |
| excalidraw | yes | — |
| garage | yes | garage-backup |
| gitea-act-runner | yes | — |
| ~~glean~~ | yes | *cron, not a timer* — **stopped and disabled 2026-08-01** |
| ingest | yes | — |
| karakeep | yes | karakeep-backup |
| kroki | yes | kroki-backup |
| metabase | yes | metabase-backup |
| n8n | yes | n8n-backup |
| openbao | yes | openbao-backup |
| pastebooks | yes | pastebooks-backup |
| pkm | yes | — |
| reader | yes | reader-backup |
| rsshub | yes | rsshub-backup |
| wallabag | yes | wallabag-backup |
| watchtower | yes | — |
| wikijs | yes | wikijs-backup |
| woodpecker-ci | yes | woodpecker-ci-backup |
| youtrack | yes | youtrack-backup, youtrack-backup-verify |

## Stacks with NO systemd unit — manual start only

These will **not** come back after a reboot without `docker compose up -d`. 20 stacks.

| Stack | RUNBOOK | Backup timer |
|---|---|---|
| argus | yes | argus-backup |
| beszel | yes | backup-beszel |
| c3x | yes | c3x-backup |
| cadvisor | yes | — |
| checkmk | yes | checkmk-backup |
| dashboard | yes | dashboard-backup |
| erugo | yes | erugo-backup |
| filebrowser | yes | backup-filebrowser |
| gatus | yes | — |
| gitea | yes | gitea-backup, gitea-backup-verify |
| glance | **no** | — |
| home_file_server | yes | — |
| kavita | yes | kavita-backup |
| lxconsole | yes | lxconsole-backup |
| pgadmin | yes | pgadmin-backup |
| picoshare | yes | backup-picoshare |
| portainer | yes | portainer-backup |
| qui | yes | backup-qui |
| rss | yes | — |
| tool-shed | yes | — |

> **`gitea` is the notable one** — it has both a RUNBOOK and a backup timer but no service unit,
> so the forge does not restart itself after a reboot.

## Stacks present but not running at 2026-07-31

| Stack | Note |
|---|---|
| filestash | Compose stack present, no unit, not running. |
| trivy | Compose stack present, no unit, not running. |
| youtrack.corrupt-20260729 | Quarantined copy of YouTrack — must not be started. |

## Stacks using `security_opt: label:disable`

SELinux confinement is bypassed for **11 stacks** — revalidate all of them after any policy update.

The names are deliberately not published in this repo. They are listed in
`PRE-UPGRADE-BASELINE-2026-07-31.local.md` (gitignored), or regenerate:

```bash
sudo grep -rl 'label:disable' /opt/containers/*/docker-compose.y*ml \
  | sed 's|/opt/containers/||;s|/docker-compose.*||'
```

## Missing documentation

- `glance` is the only stack without a `RUNBOOK.md`.

## Backup coverage

38 systemd backup/verify timers exist. Notable gaps and exceptions:

- **Glean** is backed up by cron (`/etc/cron.d/glean-backup`, 02:30 daily), not systemd — so it has
  **no catch-up** and silently skips whenever the host is down at that time (e.g. 2026-07-27).
- Stacks with no backup at all: `cadvisor`, `excalidraw`, `gatus`, `gitea-act-runner`,
  `home_file_server`, `ingest`, `pkm`, `rss`, `tool-shed`, `watchtower` — most are stateless or
  rebuild from config, but this has not been formally reviewed.
- Six timers pull backups from other k-fed hosts rather than backing up local containers:
  `backup-core`, `backup-from-web1-to-local`, `backup-web1-openbao`,
  `backup-donetick-from-web1-to-local`, `backup-matomo-from-mail1-to-local`,
  `backup-unclutter-from-web1-to-local`.

## Containers NOT in /opt/containers

- **Kasm Workspaces 1.18.1** at `/opt/kasm` — `kasm_proxy`, `kasm_rdp_https_gateway`,
  `kasm_rdp_gateway`, `kasm_agent`, `kasm_manager`, `kasm_api`, `kasm_guac`, `kasm_db`.
  **All exited; `kasm.service` disabled.**
- **pcm-ingest**, **pcm-perlite**, **pcm-perlite-web** — PCM vault stack
- **incident-dev-postgres**, **incident-valkey** — dev containers, referenced by image ID only
- **buildx_buildkit_builder0** — Docker buildx builder

## Pending changes

- **Glean is stopped and disabled** as of 2026-08-01, with all data retained. The decommission is
  tracked in `/opt/containers/TODO.md`; the open question is whether its stored articles need
  keeping. When it is finally torn down the stack count drops from 44 to 43.
- **Kasm's fate is undecided** — it has been down since before the F43 upgrade.
