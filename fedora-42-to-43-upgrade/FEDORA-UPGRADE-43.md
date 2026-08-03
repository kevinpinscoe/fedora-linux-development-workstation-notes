# Fedora 42 → 43 Upgrade Checklist

**Target:** morning of 2026-08-01  
**Host:** kevin.kevininscoe.com (Fedora 42 Adams, kernel 6.19.14-108.fc42.x86_64)  
**Desktop:** KDE Plasma 6 (plasmashell 6.6.4)  
**GPU:** AMD Radeon PRO WX 7100 (amdgpu, open-source only)

> **Refreshed 2026-07-31** against the live host. The original checklist was researched in April 2026
> and had drifted badly — container counts, kernel expectations, and the Docker packaging were all wrong.
> A pre-upgrade snapshot lives in **[`PRE-UPGRADE-BASELINE-2026-07-31.md`](PRE-UPGRADE-BASELINE-2026-07-31.md)**.
> Compare against it during QA so pre-existing faults are not mistaken for upgrade damage.
>
> This repository is public, so the port map and the list of SELinux-unconfined stacks are **not**
> published. They live on the host in `PRE-UPGRADE-BASELINE-2026-07-31.local.md` (gitignored).
> **Use the `.local.md` copy when actually running the QA pass** — it is the complete one.

## Decisions taken before this upgrade (2026-07-31)

| Question | Decision |
|---|---|
| Target release, given F42 is EOL and F44 is out | **Fedora 43**, as originally planned |
| Salt is broken on F43 (see the Salt section below) | **Resolved by removal** — Salt was uninstalled from this host on 2026-08-01 |
| Glean, pending decommission | **Stopped and disabled** on 2026-08-01, data retained — out of the upgrade's blast radius |

### Two facts that changed since the April research

1. **Fedora 42 is EOL.** Its updates repo has moved to `archives.fedoraproject.org`. The host has
   been running without updates, and Phase 1.1 below is now largely a no-op.
2. **Fedora 44 is released.** F43 is therefore N-1 with a limited support window — expect to plan a
   43 → 44 upgrade before long.

---

## Phase 1 — Pre-Upgrade Preparation

### 1.1 System Update
> **F42 is EOL.** Its updates repo is archived, so this step will find little or nothing. Run it
> anyway to confirm the package database is consistent, but do not treat "no updates" as a problem.

- [ ] Attempt a final F42 update: `sudo dnf upgrade --refresh`
- [ ] Reboot after kernel updates if any were installed
- [ ] Confirm running kernel matches latest installed: `uname -r` vs `rpm -q kernel`
  - Baseline: running `6.19.14-108.fc42`; installed `6.19.8-100`, `6.19.13-100`, `6.19.14-108`

### 1.2 Resolve Known Issues Before Upgrading
- [x] **Woodpecker-CI agent restart loop** — FIXED 2026-04-26. Root causes: `v3` tag pointed to broken `next` dev build, `WOODPECKER_BACKEND` not set, SELinux blocked Docker socket. Fixed by pinning to `v3.7.0`, adding `WOODPECKER_BACKEND: docker`, and `security_opt: label:disable` on the agent.
- [ ] Check for any other unhealthy containers: `docker ps --filter status=exited`
  - Expect the 8 **Kasm** containers to already be exited — Kasm has been down and `kasm.service`
    disabled since before the upgrade. That is the baseline state, not a fault.

### 1.2a Docker Engine — ALREADY SATISFIED, but the packaging is not what this checklist assumed

**Correction (2026-07-31):** this host does **not** run Docker CE. It runs Fedora's own packages:

| Package | Installed | In F43 |
|---|---|---|
| `moby-engine` | 29.4.2-1.fc42 | **29.6.2-1.fc43** |
| `docker-cli` | 29.4.2-1.fc42 | 29.6.2-1.fc43 |
| `docker-compose` | 5.1.2-1.fc42 | 5.3.1-1.fc43 |
| `containerd.io` | 2.2.4-1.fc42 (Docker repo) | 2.2.6-1.fc43 |

`docker-ce` and `docker-ce-cli` are **not installed** — the old command in this section
(`sudo dnf upgrade docker-ce docker-ce-cli containerd.io`) would have done nothing useful. Do not run it.

The v29 requirement is already met (29.4.2), and F43 keeps the engine on 29.x, so the
kernel-iptables concern is resolved by the upgrade itself rather than by manual action.

```bash
docker --version              # baseline: 29.4.2 — already >= 29
sudo systemctl status docker
```

- [x] Docker engine is on v29.x — verified 2026-07-31, `moby-engine-29.4.2-1.fc42`
- [ ] Docker Engine starts cleanly after the upgrade (confirm it landed on 29.6.2-1.fc43)

### 1.3 Identify Potentially Conflicting Packages
- [x] **Wine** — verified 2026-07-31: not installed. Nothing to remove.
- [x] **Third-party repo readiness** — verified 2026-07-31. RPM Fusion (free + nonfree), the Docker
  repo, and all five COPRs (`dejan/lazygit`, `ilyaz/LACT`, `jdxcode/mise`, `lihaohong/yazi`,
  `phracek/PyCharm`) all publish for F43. Tailscale, 1Password, VS Code, Cursor, LibreWolf, Grafana
  and `kevinpinscoe` use release-agnostic paths. Full table in the baseline file.
- [x] **akmods / kernel modules** — verified 2026-07-31: only `kmod` / `kmod-libs` (base system).
  No akmod-built out-of-tree modules, and no NVIDIA driver despite the RPM Fusion NVIDIA repo being
  enabled. Nothing here blocks the upgrade.
- [ ] Re-check for stragglers immediately before downloading:
  ```
  sudo dnf list extras
  ```

### 1.4 Snap Pre-Checks
- [ ] Note installed snaps: `snap list`
  - Baseline (2026-07-31): bare, core, core20, core22, core24, ffmpeg-2404, gnome-42-2204,
    gnome-46-2404, gtk-common-themes, mesa-2404, obs-studio, shortwave, snapd
  - Host `snapd` RPM is 2.75.2; the `snapd` snap itself is 2.76.1
- [ ] Snaps are self-contained — no action required, but verify they work after upgrade

---

## Phase 2 — Backups

> Docker overlay2 and volumes are **excluded** from backup.sh. Container data must be snapshotted separately.

### 2.1 Run System Backup
- [ ] Run backup manually and verify success:
  ```
  sudo ~/bin/backup.sh
  ```
- [ ] Verify mirrors are current:
  - `/root_backup` (nvme1n1p1)
  - `/boot_backup` (nvme1n1p4)
  - `/home_backup` (sdb1)

### 2.2 Run All Container Backup Timers Manually

> **Regenerated 2026-07-31 from the live host.** There are now **38** backup timers, not the 14 this
> section used to list. Most fire between 00:00 and 06:00, so if the upgrade starts early enough the
> overnight run will already have happened — check `systemctl list-timers` before firing them all by
> hand.

Local container/application backups:

- [ ] `sudo systemctl start actualbudget-backup.service`
- [ ] `sudo systemctl start argus-backup.service`
- [ ] `sudo systemctl start backup-beszel.service`
- [ ] `sudo systemctl start backup-filebrowser.service`
- [ ] `sudo systemctl start backup-picoshare.service`
- [ ] `sudo systemctl start backup-qui.service`
- [ ] `sudo systemctl start backupmysql.service`
- [ ] `sudo systemctl start c3x-backup.service`
- [ ] `sudo systemctl start checkmk-backup.service`
- [ ] `sudo systemctl start convertx-backup.service`
- [ ] `sudo systemctl start dashboard-backup.service`
- [ ] `sudo systemctl start erugo-backup.service`
- [ ] `sudo systemctl start garage-backup.service`
- [ ] `sudo systemctl start gitea-backup.service`
- [ ] `sudo systemctl start karakeep-backup.service`
- [ ] `sudo systemctl start kavita-backup.service`
- [ ] `sudo systemctl start kroki-backup.service`
- [ ] `sudo systemctl start lxconsole-backup.service`
- [ ] `sudo systemctl start metabase-backup.service`
- [ ] `sudo systemctl start n8n-backup.service`
- [ ] `sudo systemctl start openbao-backup.service`
- [ ] `sudo systemctl start pastebooks-backup.service` (includes MySQL dump)
- [ ] `sudo systemctl start pgadmin-backup.service`
- [ ] `sudo systemctl start portainer-backup.service`
- [ ] `sudo systemctl start reader-backup.service`
- [ ] `sudo systemctl start rsshub-backup.service`
- [ ] `sudo systemctl start wallabag-backup.service`
- [ ] `sudo systemctl start wikijs-backup.service`
- [ ] `sudo systemctl start woodpecker-ci-backup.service`
- [ ] `sudo systemctl start youtrack-backup.service`

Not a systemd timer — run by hand:

- [x] `sudo bash /opt/containers/glean/backup-glean.sh` — **DONE 2026-08-01 08:18**.
  `/home/backups/glean/glean-20260801-081833.sql.gz`, 144 MB, gzip verified, 42,776 lines.
  Glean's backup is a **cron** job (`/etc/cron.d/glean-backup`, 02:30 daily), not a systemd unit, so
  it has no catch-up and is silently skipped if the host is down at 02:30 — as happened 2026-07-27.
  **Glean was then stopped and disabled** (`systemctl disable --now glean.service`); its unit file
  and all eight Docker volumes are retained. Remaining teardown: `/opt/containers/TODO.md`.

Off-host pulls and verifiers — lower priority for the upgrade, but note their state:

- `backup-core`, `backup-from-web1-to-local`, `backup-web1-openbao`,
  `backup-donetick-from-web1-to-local`, `backup-matomo-from-mail1-to-local`,
  `backup-unclutter-from-web1-to-local` — pull backups from other k-fed hosts
- `gitea-backup-verify`, `youtrack-backup-verify` — verification timers

Verify recent timestamps:
```
systemctl list-timers --all | grep -E 'backup|verify'
```
- [ ] All expected timers show a recent last-run and a valid next-trigger

### 2.3 Snapshot /opt/containers Tree
This is NOT covered by backup.sh and contains all compose files, configs, and secrets.

> **CORRECTED 2026-08-01 — do NOT write snapshots to `/home_backup`.** This section used to target
> `/home_backup`, which is **wrong and silently destructive**: `~/bin/backup.sh` mirrors
> `/home/ → /home_backup/` with `--delete` and no excludes, so anything authored in `/home_backup`
> that does not exist under `/home` is erased on the next backup run. A snapshot taken there was
> confirmed deleted within minutes. Write to **`/home/backups/`** instead — it lives on sda1 and is
> then mirrored to `/home_backup` automatically, protecting it on both disks.

```bash
sudo rsync -a --delete --info=stats2 /opt/containers/ /home/backups/containers-snapshot-$(date +%Y%m%d)/
```
- [x] Snapshot completed 2026-08-01 — `/home/backups/containers-snapshot-20260801`, 16 GB,
      55 entries, 27 `.env` files present; gitea and openbao compose files spot-checked

### 2.4 Snapshot Kasm
Kasm is already stopped (all 8 containers exited, `kasm.service` disabled), so the snapshot is
consistent by default — but take it anyway, since Phase 6 may end in a reinstall or removal.
Same `/home/backups` correction as 2.3 applies.
```bash
sudo rsync -a --delete /opt/kasm/ /home/backups/kasm-snapshot-$(date +%Y%m%d)/
```
- [x] Kasm snapshot completed 2026-08-01 — `/home/backups/kasm-snapshot-20260801`, 3.4 GB

### 2.5 Retain Extra Kernels in GRUB
Raising `installonly_limit` means the known-good F42 kernels survive as fallback entries after the
upgrade. `/boot` measures ~103 MB per kernel set, so 5 kernels ≈ 825 MB of its 1.5 GB — it fits.

```bash
echo 'installonly_limit=5' | sudo tee -a /etc/dnf/dnf.conf
```
- [x] `installonly_limit=5` set 2026-08-01
- [ ] GRUB shows multiple kernel entries at boot (the F42 kernels are the fallback if F43's kernel
      misbehaves — see Phase 3.4)

---

## Phase 3 — The Upgrade

### 3.1 Upgrade Plugin — NOT NEEDED on this host

> **Corrected 2026-08-01.** `dnf-plugin-system-upgrade` is the **DNF 4** way. This host runs DNF 5,
> where `system-upgrade` is a **built-in subcommand** — there is no plugin to install, and
> `dnf5-plugin-system-upgrade` does not exist as a package.

```bash
sudo dnf system-upgrade --help   # confirms: clean | log | reboot | status | download
```
- [x] Verified 2026-08-01 — `dnf5 system-upgrade` is available with no install required

### 3.2 Download Fedora 43 Packages
```
sudo dnf system-upgrade download --releasever=43
```
- [ ] Download completed without fatal errors
- [ ] If dependency conflicts appear, resolve them (see troubleshooting below before re-running)

### 3.3 Reboot Into Upgrade
```
sudo dnf system-upgrade reboot
```
- [ ] System reboots into upgrade environment
- [ ] Upgrade completes (may take 20–40 minutes)
- [ ] System boots into Fedora 43

### 3.4 First Boot Into F43

> **Kernel expectation corrected 2026-07-31.** This section used to say "expect 6.17.x". That was
> true when F43 was released; it is not true now. F43's updates repo currently ships
> **kernel 7.1.5-101.fc43**, and `dnf system-upgrade download` pulls from updates as well as the
> release repo — so this host goes from **6.19.14 straight to 7.1.x in one hop**.
>
> The practical consequence: the amdgpu 6.17.9 page-fault regression documented further down is
> **no longer the relevant risk** — you skip that kernel range entirely. The new unknown is a
> major-version kernel jump (6.19 → 7.1) on a GCN4-era Radeon PRO WX 7100. This has not been
> researched. Treat the first hours on F43 as a GPU-stability watch (QA-12), and keep the F42
> kernels in GRUB as the fallback.

- [ ] Confirm OS version: `cat /etc/fedora-release`
- [ ] Confirm kernel: `uname -r` — expect **7.1.x-NNN.fc43.x86_64** (record the exact version)
- [ ] Confirm desktop (KDE Plasma) is working
- [ ] Confirm the F42 kernels (`6.19.14-108.fc42` and older) are still listed in GRUB as fallback

---

## Phase 4 — Post-Upgrade Verification

### 4.1 DNF 5 Migration Check
F43 ships DNF 5 (replaces DNF4/YUM). History DB is separate — packages installed via dnf4 won't appear in dnf5 history.
- [ ] `dnf --version` (should show 5.x)
- [ ] `dnf upgrade --refresh` to pick up any post-upgrade updates
- [ ] `dnf autoremove` to clean up orphaned packages
- [ ] Verify dnf-automatic or any automated update timers still work:
  ```
  systemctl status dnf-automatic.timer 2>/dev/null || echo "not in use"
  ```

### 4.2 System Services
- [ ] Check for failed systemd units: `systemctl --failed`
- [ ] Verify Apache/httpd: `sudo systemctl status httpd`
- [ ] Verify Tailscale: `sudo systemctl status tailscaled && tailscale status`
- [ ] Verify certbot renewal timer: `systemctl status certbot-renew.timer`
- [ ] Verify snapd: `sudo systemctl status snapd && snap list`

### 4.3 SELinux
- [ ] SELinux still in enforcing mode: `getenforce`
- [ ] Check for new AVC denials: `sudo ausearch -m avc -ts recent | tail -30`
- [ ] If denials appear for container services, audit and update policies as needed

---

## Phase 5 — Container Verification

### 5.1 Restart All Containers

> **Important (found 2026-07-31):** only about half the stacks have a systemd unit. The rest were
> started with `docker compose up` directly and **will not come back on their own after the reboot**.
> The old loop below silently skipped them.

**Stacks with an enabled systemd unit** — these come back by themselves; restart if needed:

```bash
for n in actualbudget convertx excalidraw garage gitea-act-runner ingest karakeep \
         kroki metabase n8n openbao pastebooks pkm reader rsshub wallabag watchtower \
         wikijs woodpecker-ci youtrack; do
  sudo systemctl restart "$n" 2>/dev/null || echo "no unit: $n"
done
```

**Stacks with NO systemd unit** — must be brought up by hand:

```bash
for n in argus beszel c3x cadvisor checkmk dashboard erugo filebrowser gatus gitea glance \
         home_file_server kavita lxconsole pgadmin picoshare portainer qui rss tool-shed; do
  (cd "/opt/containers/$n" && sudo docker compose up -d)
done
```

- [ ] All unit-managed stacks restarted
- [ ] All unit-less stacks brought up manually
- [ ] Container count is back to **67 running** — the baseline was 73, minus Glean's six, which
      were deliberately taken down on 2026-08-01 and must stay down
- [ ] Check for exited containers: `docker ps --filter status=exited`
      (the 8 Kasm containers are expected to stay exited — see Phase 6)

> `gitea` has a backup timer but no service unit. Worth deciding after the upgrade whether that is
> intentional; it means Gitea does not survive a reboot unattended.

### 5.2 Per-Container Checks

> **Regenerated 2026-07-31.** The fleet is **44 Compose stacks / 73 running containers** at baseline
> (67 after Glean was taken down on 2026-08-01), not the 17
> this table used to list. Port bindings are deliberately not published here — they are in
> `PRE-UPGRADE-BASELINE-2026-07-31.local.md`, or run `docker ps --format '{{.Names}}\t{{.Ports}}'`.

| Stack | Check | Status |
|-------|-------|--------|
| actualbudget | UI reachable | |
| argus | UI reachable | |
| beszel | UI reachable; agent reporting | |
| c3x | API reachable; db + scraper up | |
| cadvisor | UI reachable | |
| checkmk | UI reachable | |
| convertx | UI reachable | |
| dashboard | homepage reachable + socket-proxy up | |
| erugo | UI reachable | |
| excalidraw | UI reachable | |
| filebrowser | UI reachable | |
| filestash | stack present but **not running at baseline** — confirm intent | |
| garage | `garage status` healthy; admin + S3 ports reachable | |
| gatus | UI reachable | |
| gitea | UI reachable; SSH port answers | |
| gitea-act-runner | runner + dind both up, registered | |
| glance | UI reachable | |
| ~~glean~~ | **Stopped and disabled 2026-08-01** — must NOT come back up | |
| home_file_server | UI reachable | |
| ingest | unit active; check its logs | |
| karakeep | UI reachable; meilisearch + chrome up | |
| kavita | UI reachable | |
| kroki | Renders a test diagram | |
| lxconsole | UI reachable | |
| metabase | UI reachable; postgres healthy | |
| n8n | UI reachable | |
| openbao | API reachable; **must unseal with 3 keys after restart** | |
| pastebooks | UI reachable; mysql healthy | |
| pgadmin | UI reachable | |
| picoshare | UI reachable | |
| pkm | perlite web reachable | |
| portainer | UI reachable | |
| qui | UI reachable | |
| reader | UI reachable; meilisearch, flaresolverr, rss-bridge up | |
| rss | nginx reachable | |
| rsshub | Returns feed data | |
| tool-shed | UI reachable | |
| trivy | stack present but **not running at baseline** — confirm intent | |
| wallabag | UI reachable; db, redis, worker up | |
| watchtower | container up; check it is not auto-updating anything unwanted | |
| wikijs | UI reachable | |
| woodpecker-ci | Server + agent both healthy | |
| youtrack | UI reachable | |

Also present but not part of the fleet check:
- `youtrack.corrupt-20260729` — quarantined copy, must **not** be started
- `pcm-ingest` / `pcm-perlite` — run outside `/opt/containers`
- `incident-dev-postgres`, `incident-valkey` — dev containers
- `buildx_buildkit_builder0` — buildx builder

### 5.3 SELinux Labels — Containers Using `label:disable`

> **Corrected 2026-07-31:** this is **11 stacks**, not the 4 previously listed. These bypass SELinux
> confinement and are the most likely to break after an SELinux policy update.
>
> The stack names are kept out of this public repo — they are in
> `PRE-UPGRADE-BASELINE-2026-07-31.local.md`, or regenerate the list:

```bash
sudo grep -rl 'label:disable' /opt/containers/*/docker-compose.y*ml \
  | sed 's|/opt/containers/||;s|/docker-compose.*||'
```

- [ ] Every stack in that list starts cleanly
- [ ] Each one's core function works (search indexes, workflow runs, data directories readable,
      the CI agent can reach the Docker socket)
- [ ] Two of them were not running at baseline — confirm whether that is still intended
- [ ] Check audit log for new denials: `sudo ausearch -m avc -c docker -ts today`

### 5.4 Backup Timers — Confirm Still Active
```
systemctl list-timers --all | grep -E 'backup|verify'
```
- [ ] All **38** backup/verify timers are loaded and scheduled (full list in the baseline file)
- [ ] `/etc/cron.d/glean-backup` — still installed, but Glean is stopped, so it now dumps a dead
      database and will start failing. Removing it is a task in `/opt/containers/TODO.md`.

---

## Phase 6 — Kasm (Currently Down — Decision Needed)

Kasm Workspaces 1.18.1 lives at `/opt/kasm` and is NOT managed by dnf or docker-compose.

> **State change found 2026-07-31:** Kasm is **not running and has not been for days**. All 8
> containers are exited (`kasm_proxy`, `kasm_rdp_https_gateway`, `kasm_rdp_gateway`, `kasm_agent`,
> `kasm_manager`, `kasm_api`, `kasm_guac`, `kasm_db` — several with non-zero exit codes going back
> 7 days) and `kasm.service` is **disabled and inactive**.
>
> This removes Kasm as an upgrade risk: there is nothing running to break. It also means the
> post-upgrade QA-10 checks would fail for reasons that have nothing to do with F43.

- [ ] **Decide Kasm's fate** — is it being retired, or does it need to come back? Nothing else in
      this checklist can be answered until that is settled.
- [ ] If retiring: plan the teardown (ports, DNS, vhosts, service catalog) the same way as the
      Glean decommission in `/opt/containers/TODO.md`
- [ ] If keeping: check F43 compatibility at https://www.kasmweb.com/docs/latest/release_notes.html,
      check for a newer release than 1.18.1, and follow Kasm's official upgrade procedure
- [ ] If keeping, after the upgrade verify services are healthy:
  ```
  sudo /opt/kasm/bin/utils/usage_stats.py
  docker ps | grep kasm
  ```

---

---

## Potential Issues — Published Reports (researched 2026-04-26)

> **Blocker assessment (2026-04-26):** None of the issues below are upgrade blockers. All have either a shipped fix or a confirmed workaround. See individual entries for rationale.
>
> **Re-assessment 2026-07-31:** the first two entries below are now **historical**. F43's kernel has
> moved on from 6.17 to **7.1.5**, so neither the Docker/6.17-iptables issue nor the amdgpu 6.17.9
> page-fault regression can be hit on this upgrade. They are kept for the record. The live concerns
> The Salt entry further down was resolved on 2026-08-01 by removing Salt from the host outright, so
> the only live concern is the **unresearched 6.19 → 7.1 kernel jump** (Phase 3.4).

### ~~BLOCKER~~ ~~CRITICAL~~ HISTORICAL — Docker CE + kernel 6.17 iptables — **cannot occur: F43 now ships kernel 7.1.5, and this host is already on engine 29.x**

**Risk level at the time: HIGH** (the whole Compose fleet depends on Docker Engine)

**Why it no longer applies (2026-07-31):** this host runs Fedora's `moby-engine` 29.4.2 — already
past the v28 cutoff — and F43 upgrades it to 29.6.2. The 6.17 kernel range is skipped entirely.
Note also that the fix text below refers to `docker-ce` packages that are **not installed here**;
see the corrected Phase 1.2a.

Kernel 6.17 removed legacy `ip_tables` kernel modules. Docker Engine v28.x and earlier require these modules and **will fail to start** after the upgrade. Error seen:

```
bridge: filtering via arp/ip/ip6tables is no longer available by default.
Update your scripts to load br_netfilter if you need this.
```

**Fix:** Docker CE v29.x adds support for the new kernel netfilter model. Before upgrading Fedora, ensure Docker CE is updated to v29.x while still on F42:

```bash
sudo dnf upgrade docker-ce docker-ce-cli containerd.io
docker --version   # must show 29.x or later before proceeding
```

After the F43 upgrade, if Docker fails to start, load `br_netfilter` manually as a workaround:

```bash
sudo modprobe br_netfilter
sudo systemctl start docker
```

To make it persistent: `echo 'br_netfilter' | sudo tee /etc/modules-load.d/br_netfilter.conf`

Issue trackers:
- [moby/moby #50615](https://github.com/moby/moby/issues/50615) — primary upstream issue tracking kernel 6.17 + ip_tables removal; monitor for fix status and which Docker version closes it
- [coreos/fedora-coreos-tracker #1998](https://github.com/coreos/fedora-coreos-tracker/issues/1998) — Fedora-side tracking of kernel 6.17 breaking moby/Docker test suite
- [docker/for-linux #1545](https://github.com/docker/for-linux/issues/1545) — Fedora 43 repo availability tracking

Discussion: [Heads Up - Docker and F43 (Fedora Discussion)](https://discussion.fedoraproject.org/t/heads-up-docker-and-f43/161706)

---

### ~~BLOCKER~~ ~~HIGH~~ HISTORICAL — AMD amdgpu page faults and crashes (kernel 6.17.9+) — **cannot occur: F43 now ships kernel 7.1.5, skipping the affected range**

> **Superseded 2026-07-31.** The 6.17.9–6.18.3 regression window is no longer reachable — F43's
> updates repo ships **kernel 7.1.5-101.fc43**, and the host is coming from 6.19.14. The *new*
> and unresearched risk is the 6.19 → 7.1 major-version jump on this GCN4-era card. See Phase 3.4
> and QA-12; the fallback procedure below still applies, except that the kernel you fall back to is
> `6.19.14-108.fc42`, not 6.17.8.

**Risk level at the time: HIGH for this system** (AMD Radeon PRO WX 7100, amdgpu driver)

Multiple confirmed reports of amdgpu page faults (`GCVM_L2_PROTECTION_FAULT_STATUS`) starting with kernel 6.17.9. Kernel 6.17.12 has additional crash/reset reports. The transition point is 6.17.8 → 6.17.9.

- GPU crashes/resets reported at least once daily on some hardware
- GNOME desktop freezing every 10–20 min (less relevant — KDE desktop here)
- Page faults triggered by GPU-intensive workloads

**Why not a blocker:** You are already running kernel 6.19.13 on F42 with no GPU issues. The page fault regression affects kernels 6.17.9–6.18.3; fixes were backported and confirmed resolved by 6.18.4 and 6.19.x. F43 will briefly use a 6.17.x kernel during the upgrade boot, but `dnf upgrade` immediately afterward will pull in the current F43 kernel (6.19.x as of April 2026). The reported failures also appear primarily on newer RDNA hardware (Ryzen AI/Radeon 8060S), not older GCN4 workstation cards like the WX 7100.

**Pre-upgrade action:**
- Add `instonly_limit=5` or similar to `/etc/dnf/dnf.conf` to retain extra kernel entries in GRUB
- Confirm GRUB shows multiple kernel options at boot

**Post-upgrade mitigation:** If amdgpu instability occurs, boot previous kernel from GRUB. To pin to a known-good kernel:

```bash
sudo dnf install kernel-6.17.8-300.fc43.x86_64   # adjust version as needed
# Set as default in GRUB:
sudo grubby --set-default /boot/vmlinuz-6.17.8-300.fc43.x86_64
```

Issue tracker: No single confirmed upstream issue was found for the 6.17.9 regression specific to the WX 7100. The canonical place to monitor is [drm/amd GitLab issues](https://gitlab.freedesktop.org/drm/amd/-/issues) — search for `GCVM_L2_PROTECTION_FAULT` to find any filed reports. If instability occurs post-upgrade, file a report there with `uname -r`, `lspci | grep VGA`, and the full kernel log excerpt.

Discussion:
- [amdgpu page fault kernel 6.17.9 (Fedora Discussion)](https://discussion.fedoraproject.org/t/amdgpu-page-fault-on-kernel-6-17-9-300-works-on-6-17-8-300-fedora-43/175852)
- [Frequent GPU crashes kernel 6.17.12 (Fedora Discussion)](https://discussion.fedoraproject.org/t/frequent-gpu-crashes-resets-on-fedora-43-with-kernel-6-17-12/178920)

---

### ~~BLOCKER~~ MEDIUM — KDE Plasma duplicate packages during upgrade — **NOT A BLOCKER: confirmed workaround exists**

**Risk level: MEDIUM** (this system runs KDE Plasma 6)

After the F42→F43 upgrade, KDE Plasma 6.5.1 introduced duplicate package conflicts that caused upgrade failures and post-upgrade issues including `plasma-plasmashell` failing to start (black screen + cursor). `krunner` is specifically flagged.

**If upgrade fails with duplicate package errors:**

```bash
sudo dnf distro-sync --skip-broken --setopt=protected_packages=
```

**If KDE/Plasma won't start post-upgrade (black screen):**

```bash
sudo dnf downgrade krunner
reboot
```

**Pre-upgrade:** Note current Plasma version (`plasmashell --version`) so you have a baseline.

Issue tracker: No dedicated upstream KDE or Fedora Bugzilla issue was found for this specific duplicate-package regression — it was resolved via `dnf distro-sync` without a formal bug being filed. If plasma-related failures appear post-upgrade, search [bugs.kde.org](https://bugs.kde.org) or [bugzilla.redhat.com](https://bugzilla.redhat.com) for `krunner` + `Fedora 43`.

Discussion:
- [KDE Upgrade to FC43 Failed - duplicate packages](https://discussion.fedoraproject.org/t/fedora-kde-upgrade-to-fc43-failed-lots-of-duplicate-packages/170635)
- [Frequently encountered issues following Plasma 6.5.1](https://discussion.fedoraproject.org/t/frequently-encountered-issues-following-plasma-6-5-1-release-in-f43/171395)

---

### MEDIUM — Wine blocks upgrade (confirmed)

Already captured in Phase 1.3, but confirmed by multiple real upgrade reports:

```bash
rpm -q wine && sudo dnf remove wine
```

Remove wine before running `dnf system-upgrade download`. Reinstall after if needed (check RPM Fusion F43 availability first).

Source: [System Upgrade to 43 fails due to WINE](https://discussion.fedoraproject.org/t/system-upgrade-to-43-fails-due-to-wine/170400)

---

### ~~WARNING~~ RESOLVED — Salt / SaltStack broken on Fedora 43 (Python 3.14) — **Salt removed from this host 2026-08-01**

> ## Resolved by removal — read this first
>
> This entry was first a stop-condition ("do not upgrade until Salt works on Python 3.14"), then a
> waiver. Neither applies now: **Salt was removed from this host entirely on 2026-08-01**, before the
> upgrade, because it was outmoded. There is nothing left to break.
>
> What was removed: `salt`, `salt-master`, and `salt-minion` (3007.5-2.fc42), plus `/etc/salt`,
> `/var/cache/salt`, `/var/log/salt`, and the master's accepted minion key store. `dnf` also reaped
> 31 orphaned dependencies, all clean — the only notable one being `dnf-utils`, a DNF 4 compat
> package Fedora 43 drops anyway. Ports 4505/4506 are no longer listening and no salt units remain.
> A safety archive was written to `/home_backup/salt-config-backup-20260801.tar.gz` (51 KB).
>
> Nothing running was orphaned: the four minions the master knew (`dev-agents-1`, `dev-servers-1`,
> `prod-agents-1`, `prod-servers-1`) are all on VMs that are shut off. Neither 4505 nor 4506 was
> reserved in portman, so no `portman release` was needed. Logged in `~/ai/fedora/CHANGELOG.md`.
>
> **Nothing to check during QA.** The detail below is kept only as the record of why Salt went.

**Affects:** any host running `salt-minion` from the Fedora 43 repo (`salt-3007.6-3.fc43`).
**Risk level: HIGH** if you manage this host or VMs it runs as Salt minions.

Fedora 43 ships Python 3.14, which introduced breaking changes to the `multiprocessing` pickling protocol. Salt 3007.x has not been updated to handle these changes. Two distinct failure modes are present:

**Failure 1 — Missing undeclared Python dependencies.** The Fedora 43 `salt` RPM declares only `python3-tornado` as a Python dep but actually requires: `python3-looseversion`, `python3-packaging`, `python3-msgpack`, `python3-pyzmq`, `python3-cryptography`, and `python3-tornado`. Without these, `salt-minion` exits immediately on start with `ModuleNotFoundError`.

**Failure 2 — Multiprocessing pickling crash (deeper incompatibility).** Even after installing all missing deps, `salt-minion` starts and connects to the master but cannot respond to commands. It crashes with:

```
AttributeError: 'SignalHandlingProcess' object has no attribute '_args_for_getstate'
```

This is caused by Python 3.14 changing how `multiprocessing.popen_forkserver` pickles process objects. Salt's `SignalHandlingProcess` subclass does not implement `__getstate__` in a way compatible with Python 3.14. This is a Salt upstream bug — no patch in `salt-3007.6-3.fc43` at time of writing.

**Practical consequence:** `salt '*' test.ping` returns `[No response]` from any Fedora 43 minion, making the minion effectively non-functional.

**Workaround options (in order of preference):**

1. **Use Fedora 42 base images for VMs** — Salt 3007.5 on Python 3.13 works correctly.
2. **Wait for Salt upstream fix** — watch https://github.com/saltstack/salt for a Python 3.14 compat PR.
3. **Skip Salt on short-lived k3s VMs** — these are rebuilt frequently; manage them via SSH/Ansible instead.

**This host (kevin.kevininscoe.com):** ~~Do NOT upgrade this host to Fedora 43 until Salt Python
3.14 compat is confirmed working.~~ **Moot — Salt was removed on 2026-08-01.** See the box at the
top of this section.

---

### LOW — Snap / squashfs on F43

Snapd is packaged for F43 and generally works (this host is on `snapd-2.75.2` at baseline, with the
`snapd` snap itself at 2.76.1 — both well past the 2.72 originally noted here). One known failure
mode: snap apps fail to mount if `fuse` / `squashfuse` packages are missing.

**Post-upgrade check if snaps fail:**

```bash
sudo dnf install fuse squashfuse
sudo systemctl restart snapd
snap list
```

Installed snaps to verify: obs-studio, shortwave, ffmpeg-2404, core20/22/24, gnome-42/46-2204/2404.

Source: [Snap stopped working on Fedora Discussion](https://discussion.fedoraproject.org/t/latest-fedora-update-broke-snap-and-flatpak-apps/154650)

---

## Troubleshooting — Known Upgrade Issues

### RPM Fusion conflicts
```
# Temporarily disable RPM Fusion to diagnose:
sudo dnf system-upgrade download --releasever=43 --disablerepo='rpmfusion*'
# Then re-enable after upgrade and run: sudo dnf distro-sync
```

### Qt6/Dolphin/KDE conflicts
- If KDE packages conflict, try: `sudo dnf system-upgrade download --releasever=43 --allowerasing`
- Be cautious — `--allowerasing` can remove packages; review what it plans to remove

### Kernel boot failure
- At GRUB, select an older kernel to boot — the retained **F42** kernels (`6.19.14-108.fc42` and
  older) are the fallback, since F43 goes straight to 7.1.x
- After a stable boot: `sudo grubby --set-default /boot/vmlinuz-6.19.14-108.fc42.x86_64` to pin it
- Do not let `dnf autoremove` or a low `installonly_limit` reap the F42 kernels until the F43 kernel
  has proven stable

### General dependency conflicts
```
sudo dnf system-upgrade download --releasever=43 --allowerasing
```
Review the package list carefully before confirming.

---

## After Upgrade QA

A systematic QA pass to run after the system is back up and containers are restarted. Steps marked **[AI]** can be run by an AI agent via shell commands. Steps marked **[HUMAN]** require visual/interactive verification.

> **Compare every result against [`PRE-UPGRADE-BASELINE-2026-07-31.md`](PRE-UPGRADE-BASELINE-2026-07-31.md).**
> Several things were already broken before the upgrade; without the baseline they read as new damage.

### QA-1 — OS and Kernel

**[AI]**
```bash
cat /etc/fedora-release                          # expect: Fedora release 43
uname -r                                         # expect: 7.1.x-NNN.fc43.x86_64
rpm -q kernel | sort                             # confirm F42 kernels retained as fallback
rpm --version                                    # expect: RPM version 6.x (baseline was 4.20.1)
dnf --version                                    # expect: 5.x (already 5.2.18 on F42 — not a real signal)
getenforce                                       # expect: Enforcing
systemctl --failed --no-pager                    # compare against the baseline list below
```

- [ ] OS is Fedora 43
- [ ] Kernel is **7.1.x** (note exact version — this is a major-version jump from 6.19; see Phase 3.4)
- [ ] F42 kernels still listed in GRUB as fallback
- [ ] RPM 6.x confirmed (this is the meaningful package-stack change; DNF was already 5.x)
- [ ] SELinux Enforcing
- [ ] **Failed units — do not expect zero.** Already failing immediately before the upgrade
      (revised 2026-08-01): `check-ai-skill-jobs`, `check-changelog-roll`, `dnsmasq`,
      `github-poll-for-activity`, ~10 `drkonqi-coredump-processor@*` instances, and a transient
      `drill-execstartpost-*` test unit. `check-pcm-nightly-ingest` and `gitea-backup` were both
      fixed on 2026-08-01 and **should not** reappear. Only **new** names are upgrade damage.

### QA-2 — Core System Services

**[AI]**
```bash
sudo systemctl status httpd --no-pager           # Apache reverse proxy
sudo systemctl status tailscaled --no-pager      # Tailscale
sudo systemctl status certbot-renew.timer --no-pager
sudo systemctl status snapd --no-pager
sudo systemctl status docker --no-pager
docker --version                                 # expect 29.6.2 (from moby-engine, not docker-ce)
rpm -q moby-engine docker-cli docker-compose containerd.io
rpm -qa | grep -i '^salt' || echo 'salt absent — removed 2026-08-01, expected'
```

- [ ] httpd running
- [ ] Tailscale connected: `tailscale status`
- [ ] certbot-renew.timer active and scheduled
- [ ] snapd running
- [ ] Docker Engine running, `moby-engine` on 29.6.2-1.fc43
- [ ] Salt still absent (removed pre-upgrade — nothing to check)

### QA-3 — SELinux AVC Denials

**[AI]**
```bash
sudo ausearch -m avc -ts today --no-pager 2>/dev/null | tail -40
sudo ausearch -m avc -ts today -c docker --no-pager 2>/dev/null | tail -20
```

- [ ] No unexpected AVC denials for container services
- [ ] No denials for Docker socket access
- [ ] If denials exist: note the `scontext` and `tcontext`, check relevant RUNBOOK

### QA-4 — Docker and All Containers

**[AI]**
```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}' | sort
docker ps --filter status=exited --format 'table {{.Names}}\t{{.Status}}'
```

- [ ] Container count back to **67 running** (baseline 73, less Glean's six — down by design)
- [ ] All expected containers show `Up` — none unexpectedly in `Exited` or `Restarting`
- [ ] The 20 unit-less stacks were brought up by hand (Phase 5.1) — they do **not** auto-start
- [ ] All 11 `label:disable` stacks running (list in the `.local.md` baseline — see Phase 5.3)
- [ ] Expected exceptions, all by design: the 8 Kasm containers stay exited, Glean's six stay down,
      and `youtrack.corrupt-20260729` stays down

### QA-5 — Container Health Endpoints

**[AI]** — the old version of this section covered 12 of 44 stacks. Sweep every published port
instead, then follow up on the service-specific endpoints below.

```bash
# Sweep every HTTP endpoint the fleet is currently publishing, derived from Docker itself
docker ps --format '{{.Names}}\t{{.Ports}}' \
  | grep -oE '127\.0\.0\.1:[0-9]+|0\.0\.0\.0:[0-9]+' \
  | cut -d: -f2 | sort -un \
  | while read -r p; do
      printf '%-6s %s\n' "$p" "$(curl -s -o /dev/null -m 10 -w '%{http_code}' "http://127.0.0.1:$p")"
    done
```

A `2xx`, `3xx`, or even `401`/`403` means the service is up; `000` means nothing is listening.

Compare the result against the port map in **`PRE-UPGRADE-BASELINE-2026-07-31.local.md`** — 40 ports
were published at baseline. That map is deliberately kept out of the published repo, so the command
above derives the list live rather than hardcoding it.

Service-specific checks that a status code alone will not catch:

```bash
# openbao — must show "sealed":false (SEALS ON EVERY RESTART)
curl -s https://openbao.kevininscoe.com/v1/sys/health | python3 -m json.tool

# garage
sudo docker exec garage garage status

# gitea — API version, plus the SSH port
curl -sf https://git.kevininscoe.com/api/v1/version && echo OK
ssh -T -p 2223 git@git.kevininscoe.com 2>&1 | head -1

# rsshub — must return feed data, not just 200
curl -sf https://rsshub.kevininscoe.com/healthz && echo OK

# woodpecker-ci — server health, then confirm the agent registered
WP=$(docker port woodpecker-server 8000 | head -1)
curl -sf "http://$WP/healthz" && echo OK
sudo docker logs woodpecker-agent --tail 20 2>&1 | grep -i 'connect\|error'

# youtrack — expect 200 only after migration completes (60-90s)
YT=$(docker port youtrack 8080 | head -1)
curl -sf -o /dev/null -w "%{http_code}" "http://$YT"

# kroki — actually render something
KR=$(docker port kroki-core 8000 | head -1)
curl -sf -o /dev/null -w "%{http_code}" "http://$KR/graphviz/svg/eNpLyUwvSizIUHBXqPZIzcnJVwjPL8pJUQQAgKQIAA=="

# databases behind the apps
for c in metabase-postgres wallabag-db wikijs-db pastebooks-db; do   # glean-postgres is down by design
  printf '%-20s %s\n' "$c" "$(sudo docker inspect -f '{{.State.Health.Status}}' $c 2>/dev/null)"
done
```

- [ ] Port sweep: every port that was listening at baseline is listening again
- [ ] OpenBao unsealed (see QA-6)
- [ ] Garage reports healthy
- [ ] Gitea API responds and its SSH port answers
- [ ] RSSHub returns feed data
- [ ] Woodpecker agent reconnected to the server
- [ ] YouTrack finished migrating and returns 200
- [ ] Kroki renders a diagram
- [ ] All backing databases report `healthy`

**[HUMAN]** — spot-check a few UIs in browser to confirm they render correctly (not just HTTP 200):
- [ ] Gitea: `https://git.kevininscoe.com` — login works, repos visible
- [ ] Woodpecker CI: `https://woodpecker-ci.kevininscoe.com` — can see pipelines
- [ ] OpenBao: `https://openbao.kevininscoe.com/ui` — can log in and read secrets

### QA-6 — OpenBao Unsealed

**[AI]**
```bash
curl -s https://openbao.kevininscoe.com/v1/sys/health | python3 -m json.tool
```

- [ ] `"initialized": true`
- [ ] `"sealed": false`
- [ ] `"standby": false`

If sealed, unseal with 3 keys (see openbao RUNBOOK.md).

### QA-7 — YouTrack Migration Complete

**[AI]** — only relevant if YouTrack image was updated as part of this upgrade:
```bash
sudo docker logs youtrack --tail 30 2>&1 | grep -E "ready|migration|error"
```

- [ ] Log contains "YouTrack is ready" (not still migrating)
- [ ] No ERROR lines in last 30 log entries

### QA-8 — Backup Scripts Still Executable

**[AI]** — SELinux policy update can revert `bin_t` labels to `container_file_t` on backup scripts, breaking systemd execution:

```bash
for f in /opt/containers/*/backup.sh; do
  label=$(ls -Z "$f" 2>/dev/null | awk '{print $1}')
  echo "$f → $label"
done
```

> **Corrected 2026-08-01 — this check is overbroad as originally written.** Only scripts a unit
> **executes directly** need `bin_t`. Units that invoke theirs as `/bin/bash <path>` need only read
> access, and work fine at `container_file_t`; 9 backup scripts sit at `container_file_t` and back up
> normally. Check the unit's `ExecStart` before "fixing" a label.
>
> This is exactly how `gitea-backup` broke on 2026-08-01: it is the only unit that directly execs a
> script under `/opt/containers`, the script was rewritten on Jul 31 and inherited
> `container_file_t` from the parent rule, and no `restorecon` followed. Note that a durable
> `semanage fcontext` rule for it **already existed** — so the fix is `restorecon`, not `chcon`, and
> plain `restorecon` will refuse (it reads the wrong label as an admin customization when the user
> prefix is `unconfined_u`). Use `-F`:
>
> ```bash
> sudo restorecon -F -v /opt/containers/<name>/<script>.sh
> ```

Scripts that a unit execs directly must show `bin_t`. If one shows `container_file_t`:

```bash
sudo chcon -t bin_t /opt/containers/<name>/backup.sh
```

- [ ] All backup.sh files have `bin_t` label

### QA-9 — Backup Timers Active

**[AI]**
```bash
systemctl list-timers --all | grep -E "backup|verify"
ls -l /etc/cron.d/glean-backup && systemctl is-active crond
```

- [ ] All **38** backup/verify timers listed and showing a next trigger time (full list in the baseline)
- [ ] No timer in `failed` state
- [ ] Glean's cron backup entry survived the upgrade and `crond` is active

Run one manual backup to confirm the full pipeline works end-to-end:
```bash
sudo systemctl start woodpecker-ci-backup.service
journalctl -u woodpecker-ci-backup.service -n 20 --no-pager
```

- [ ] Manual backup completes without error

### QA-10 — Kasm Workspaces

> **Not applicable as written.** Kasm was already **down and disabled before the upgrade** — all 8
> containers exited, `kasm.service` disabled and inactive. The checks below only apply once Phase 6's
> "decide Kasm's fate" question is answered and Kasm is deliberately brought back.

**[AI]** — confirm nothing changed unexpectedly:
```bash
sudo docker ps -a | grep kasm       # expect: still exited
systemctl is-enabled kasm.service   # expect: disabled
ls /opt/kasm                        # expect: still installed, 1.18.1
```

- [ ] Kasm still down and disabled — matches baseline, no action needed
- [ ] `/opt/kasm` intact (the snapshot from Phase 2.4 is the safety net)

Only if Kasm is being restored:
- [ ] All 8 Kasm containers `Up`, `kasm.service` active
- [ ] **[HUMAN]** Log into the Kasm web UI and launch a workspace session
- [ ] `kasm-network-plugin.service` running

### QA-11 — Snap Apps

**[AI]**
```bash
snap list
snap refresh --list 2>/dev/null || echo "up to date"
systemctl status snapd --no-pager
```

- [ ] All previously installed snaps listed: bare, core, core20, core22, core24, ffmpeg-2404,
      gnome-42-2204, gnome-46-2404, gtk-common-themes, mesa-2404, obs-studio, shortwave, snapd
- [ ] snapd running without errors

**[HUMAN]**
- [ ] Launch one snap app (e.g. shortwave) to confirm it opens

### QA-12 — AMD GPU Stability

> **Raised in priority 2026-07-31.** This upgrade jumps the kernel from 6.19.14 to **7.1.x** in one
> step, on a GCN4-era Radeon PRO WX 7100. That combination has not been researched and is now the
> single largest unknown in the upgrade. Watch it properly rather than treating it as a formality.

> **ANSWERED — the card is fine on kernel 7.1.5, and QA-12 is CLOSED.** Two soaks found **zero**
> GPU faults, ring timeouts, resets, or DRM errors: 35 minutes on the gfx ring (2026-08-02) and
> 15 minutes on the UVD/VCE video engines (2026-08-03). The biggest unresearched risk carried into
> F43 is closed, and the same answer carries forward to F44 (see
> `../fedora-43-to-44-upgrade/notes-fedora44-upgrade-planning.md`).
> The video leg required installing `mesa-va-drivers-freeworld` first — see the resolved finding
> at the end of this section.

**[HUMAN]** — monitor through at least 30 minutes of normal use, including something GPU-intensive:

```bash
# Check for GPU errors in kernel log
sudo journalctl -k | grep -i "amdgpu\|gpu.*fault\|drm.*error" | tail -20

# Confirm the driver bound and which one
lspci -k | grep -A3 -i vga
```

- [x] amdgpu bound to the card and no driver fallback to software rendering — 2026-08-02.
      `radeonsi` / POLARIS10, OpenGL **4.6** compatibility profile, Mesa 25.3.6, ACO shader
      compiler. All 9 IP blocks initialized at boot, including `uvd_v6_3_0` and `vce_v3_4_0`.
- [x] No `GCVM_L2_PROTECTION_FAULT_STATUS` or GPU reset messages in the kernel log — 2026-08-02.
      Zero across the whole 35-minute soak and the boot that preceded it.
- [x] Desktop rendering looks correct (no artifacts, no compositor restarts) — 2026-08-02.
      Three displays (DP-1, DP-3, DP-4) stayed up throughout; no compositor restart.
- [ ] If instability occurs: record `uname -r`, then reboot into **`6.19.14-108.fc42`** from GRUB —
      the F42 kernels are the fallback, not 6.17.8 as this checklist used to say

#### Soak method and results — 2026-08-02 16:00–16:35

Harness: `~/tmp/gpu-soak-20260802.sh` (not committed; it is a scratch script). It ran
`glmark2 --run-forever` and sampled temperature, power, fan, clocks and busy% every 15 s for
35 minutes, aborting early on any kernel GPU fault. `glmark2` and `radeontop` were installed from
the Fedora repo for this — CLI tools only, no unit and no config, so out of FLDW changelog scope.

| Metric | Result |
|---|---|
| Kernel GPU faults / ring timeouts / resets / VM faults | **0** |
| Duration / samples | 35 min / 140 |
| Edge temperature | mean 83.8 °C, peak **93 °C** (critical 99 °C) |
| Power (PPT) | mean 62.9 W, peak **95.0 W** (cap 95 W) |
| GPU busy | mean 54%, peak 100% |
| Fan | peak 3,617 RPM of 4,500 |
| Cooldown | 93 °C → 66 °C promptly once load was removed |

**The throttling seen is power-limited, not thermal, and is correct behaviour.** Core clock drops
off its 1243 MHz peak to 1076–1209 MHz occur precisely at the 95 W samples — the card meeting its
power cap. A failing thermal interface produces the opposite signature: temperature-limited
throttling near the 99 °C critical point with power well *under* cap. Combined with the fast
cooldown, heat transfer off the die looks healthy. **No evidence the thermal paste has degraded**,
though the card's VBIOS dates it to 2016/11/14 and there is no record anywhere in this repo, the
FLDW change log, or the Obsidian vaults of it ever having been repasted.

#### Finding — hardware video decode was unavailable (pre-existing, not upgrade damage) — FIXED 2026-08-03

The soak was to have included an `ffmpeg h264_vaapi` encode loop to exercise the VCE ring. **It
never ran** — every iteration failed instantly with `No usable encoding profile found`. Cause:

- `vainfo` exposes **only** `VAProfileMPEG2Simple`, `VAProfileMPEG2Main`, and
  `VAProfileJPEGBaseline`. No H.264, HEVC, or VP9, and **no encode entrypoints at all**.
- Verified directly: `mpv --hwdec=vaapi` on an H.264 file logs `Looking at hwdec h264-vaapi...`
  then falls back to **software decoding**.
- Installed is stock Fedora `mesa-va-drivers`, which has H.264/HEVC/VC-1 **stripped for patent
  reasons**. RPM Fusion's `mesa-va-drivers-freeworld` (available, `25.3.6-1.fc43`) supplies them
  and is **not** installed.

**This is not a hardware, driver, or card-age problem.** UVD firmware 1.130 and VCE firmware 53.26
both reported `initialized successfully` at boot. Stock Fedora produces this same MPEG2-and-JPEG
`vainfo` output on *every* GPU, including current models — it is a licensing packaging decision,
not a support decision.

**Is it upgrade damage? Probably not — but it cannot be proven either way.** RPM Fusion was
verified available for F43 *before* the upgrade (see the baseline's third-party repo table), and
the four other freeworld packages on this host — `libavcodec-freeworld`,
`gstreamer1-plugins-bad-freeworld`, `vlc-plugins-freeworld`, `libheif-freeworld` — all came through
at `fc43` versions. Had the upgrade been dropping freeworld packages as a class, those would be
gone too. `dnf history` holds no record of `mesa-va-drivers-freeworld` ever being installed. The
likeliest reading is that it was never installed and this gap long predates F43.

**The consequence while it stands:** all H.264/HEVC video on this host decodes on the CPU —
browsers, `mpv`, and Kasm Workspaces (which streams video by design) — while a working, initialized
UVD block sits idle. On a host already running 67 containers that is wasted CPU and heat, most
noticeably on 4K HEVC.

#### Resolved 2026-08-03 — codecs restored, and the video engines pass too

`mesa-va-drivers-freeworld` 25.3.6-1.fc43 installed from RPM Fusion, **both arches**:

```bash
sudo dnf install --allowerasing mesa-va-drivers-freeworld.x86_64 mesa-va-drivers-freeworld.i686
```

**Install both arches.** `dnf swap mesa-va-drivers mesa-va-drivers-freeworld` proposes removing
*both* arches of the stock package while installing only `x86_64`, which would leave 32-bit VAAPI
with no driver. Steam, Wine 11.0 and Bottles are installed here alongside a full 32-bit graphics
stack (`mesa-dri-drivers.i686`, `mesa-libGL.i686`, `mesa-vulkan-drivers.i686`, `libva.i686`), so
the `i686` driver is genuinely used. In the event nothing needed erasing — the freeworld driver
installs to `/usr/lib64/dri-freeworld/` and `libva` prefers that path, so both packages coexist.

**VAAPI profiles went 3 → 15.** Decode now covers MPEG2, VC-1, H.264 (Constrained
Baseline/Main/High), **HEVC Main, HEVC Main 10**, and JPEG. Encode came back as well — H.264 (all
three profiles) and HEVC Main — since stock Mesa strips both directions, not just decode.

**Video-engine soak — 2026-08-03 06:48–07:03, 15 minutes, 60 samples.** Both engines driven at
once: `mpv --hwdec=vaapi-copy` on a 3840x2160 HEVC Main 10 film (UVD decode) plus an
`ffmpeg h264_vaapi` encode loop (VCE encode) — the workload that could not run at all before.

| Metric | Result |
|---|---|
| Kernel GPU / UVD / VCE faults | **0** |
| Software-decode fallbacks | **0** — hardware decode held for the full run |
| Decode errors / encode errors | **0** / **0** |
| Edge temperature | mean 86.6 °C, peak 91 °C |
| Power (PPT) | mean 46.3 W, peak 49.2 W (cap 95 W) |
| Idle after | 66 °C / 25 W — back to baseline |

**Qualifier on the decode leg:** `--untimed` did not take effect as intended. The film advanced
15 minutes of content in 15 minutes of wall time, so decoding ran at **playback rate**, not
maximum throughput. This is a sustained realistic workload — which is what QA-12 asks for — but
it is not a UVD saturation test, and should not be read as one.

Note the video engines draw far less power than the gfx ring: ~46 W sustained versus the 95 W cap
the `glmark2` soak hit. Temperature still reached 91 °C, because the fan ramped only to ~1,650 RPM
here against 3,617 RPM during the gfx soak. Not a fault — worth knowing when reading GPU
temperatures on this host.

**Diagnostic trap worth recording.** `mpv --hwdec=vaapi` with a video output that cannot display
hardware frames (`--vo=null`) silently reports `Using software decoding`. That reads exactly like a
broken driver and cost real time during this investigation. Use `--hwdec=vaapi-copy` when testing
headless. Likewise, `ffmpeg` prints the base decoder name (`hevc (native)`) in its stream mapping
even when a hwaccel *is* attached — check for `Format vaapi chosen by get_format()` in
`-loglevel debug` output instead of trusting the mapping line.

**QA-12 is closed.** Both rings proven on kernel 7.1.5: gfx on 2026-08-02, UVD and VCE on
2026-08-03.

### QA-13 — Salt (removed — nothing to do)

Salt was uninstalled on 2026-08-01, before the upgrade. This step is retained only so the QA
numbering does not shift.

```bash
rpm -qa | grep -i '^salt' || echo 'salt absent — expected'
```

- [ ] Confirms absent — no further action

### QA Sign-off

- [ ] All QA-1 through QA-13 checks passed (or failures documented with workarounds applied)
      — note QA-10 (Kasm) and QA-13 (Salt) are both no-ops by design
- [ ] Date/time of completed QA: _______________
- [ ] Kernel version running: _______________
- [ ] Container count (expect 67; baseline 73 less Glean's six): _______________
- [ ] Salt absent (removed pre-upgrade): _______________
- [ ] Any open issues: _______________

---

## Post-Upgrade Notes

- [ ] Update this checklist with any issues encountered and how they were resolved
- [x] Create RUNBOOK.md files for the 7 containers that were missing them — DONE 2026-04-26 (actualbudget, excalidraw, karakeep, n8n, pastebooks, wikijs, youtrack)
- [x] Add Fedora 43 upgrade notes to existing RUNBOOK.md files — DONE 2026-04-26 (convertx, filestash, garage, gitea, glean, home_file_server, kroki, openbao, rsshub, woodpecker-ci)
- [ ] Run `sudo dnf distro-sync` to realign any 3rd-party packages to F43 equivalents
- [ ] **Decide Kasm's fate** (Phase 6) — it has been down and disabled since before the upgrade
- [x] **Salt** — removed from the host on 2026-08-01 rather than carried across the upgrade
- [ ] **Plan the 43 → 44 upgrade.** F44 is already released, so F43 is N-1 with a limited support
      window — and F44 carries the newer Salt. Do not repeat the F42 situation of sitting on an EOL
      release.
- [ ] Add systemd units for the 20 stacks that have none, or record deliberately that they are
      manual-start only (they did not survive this reboot without hand-holding)
- [ ] Resume the Glean decommission — now tracked in `/opt/containers/TODO.md`. It is stopped and
      disabled with all data intact; the open question is whether the stored articles need keeping.
- [ ] **Bump Fedora version references from 42 → 43** now that the FLDW is actually on F43. This was deliberately deferred during the 2026-07-22 `fedora/` → `FLDW/` rename because the host was still on F42; on 2026-07-22 all three source-of-truth files were aligned to F42 so they match the live host until the upgrade actually happens. Change "Fedora Linux 42" → "Fedora Linux 43" in each:
  - `~/ai/me.md` — *Systems and Environments* section.
  - `~/ai/directives/kevins-federated-unix-universe.md` — the `FLDW` row in the Home table and the `FLDW` *Host-specific parameters* Description (two spots).
  - Service catalog `~/Projects/private/fedora-dashboard/kevins-federated-unix-universe-services.md` — the `kevin.network.kevininscoe.com` FLDW fleet-host row (commit + push the `fedora-dashboard` repo per its own rule).
