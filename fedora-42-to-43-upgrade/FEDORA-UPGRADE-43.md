# Fedora 42 → 43 Upgrade Checklist

**Target:** Week of 2026-04-28  
**Host:** kevin.kevininscoe.com (Fedora 42 Adams, kernel 6.19.13-100.fc42.x86_64)  
**Desktop:** KDE Plasma 6  
**GPU:** AMD Radeon PRO WX 7100 (amdgpu, open-source only)

---

## Phase 1 — Pre-Upgrade Preparation

### 1.1 System Update
- [ ] Fully update F42 first: `sudo dnf upgrade --refresh`
- [ ] Reboot after kernel updates if any were installed
- [ ] Confirm running kernel matches latest installed: `uname -r` vs `rpm -q kernel`

### 1.2 Resolve Known Issues Before Upgrading
- [x] **Woodpecker-CI agent restart loop** — FIXED 2026-04-26. Root causes: `v3` tag pointed to broken `next` dev build, `WOODPECKER_BACKEND` not set, SELinux blocked Docker socket. Fixed by pinning to `v3.7.0`, adding `WOODPECKER_BACKEND: docker`, and `security_opt: label:disable` on the agent.
- [ ] Check for any other unhealthy containers: `docker ps --filter status=exited`

### 1.2a Upgrade Docker CE to v29.x BEFORE the Fedora Upgrade

**This is required.** Kernel 6.17 (shipped with F43) removes legacy iptables modules that Docker v28.x needs. All 17 Docker Compose services will fail to start if Docker Engine is not on v29.x.

```bash
sudo dnf upgrade docker-ce docker-ce-cli containerd.io
docker --version   # confirm 29.x before proceeding
sudo systemctl status docker
```

- [ ] Docker CE upgraded to v29.x
- [ ] Docker Engine starts cleanly

### 1.3 Identify Potentially Conflicting Packages
- [ ] Check for Wine (known upgrade conflict — remove if present):
  ```
  rpm -q wine && sudo dnf remove wine
  ```
- [ ] List RPM Fusion packages to check for F43 availability:
  ```
  sudo dnf repoquery --installed --disablerepo='*' --enablerepo='rpmfusion*' 2>/dev/null | head -40
  ```
- [ ] Check for other 3rd-party repo packages that may conflict:
  ```
  sudo dnf list extras
  ```
- [ ] Check for akmods / kernel modules that may block upgrade (NVIDIA-specific, but verify):
  ```
  rpm -qa | grep -E 'akmod|kmod'
  ```

### 1.4 Snap Pre-Checks
- [ ] Note installed snaps: `snap list`
  - Current snaps: obs-studio, shortwave, ffmpeg-2404, core20/22/24, gnome-42/46-2204/2404
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
Run each timer that has not fired recently:
- [ ] `sudo systemctl start actualbudget-backup.service`
- [ ] `sudo systemctl start karakeep-backup.service`
- [ ] `sudo systemctl start n8n-backup.service`
- [ ] `sudo systemctl start wikijs-backup.service`
- [ ] `sudo systemctl start youtrack-backup.service`
- [ ] `sudo systemctl start pastebooks-backup.service` (includes MySQL dump)
- [ ] `sudo systemctl start gitea-backup.service`
- [ ] `sudo systemctl start garage-backup.service`
- [ ] `sudo systemctl start glean-purge.service`
- [ ] `sudo systemctl start kroki-backup.service`
- [ ] `sudo systemctl start convertx-backup.service`
- [ ] `sudo systemctl start openbao-backup.service`
- [ ] `sudo systemctl start rsshub-backup.service`
- [ ] `sudo systemctl start woodpecker-ci-backup.service`

Verify recent timestamps:
```
systemctl list-timers --all | grep backup
```

### 2.3 Snapshot /opt/containers Tree
This is NOT covered by backup.sh and contains all compose files, configs, and secrets:
```
sudo rsync -av --delete /opt/containers/ /home_backup/containers-snapshot-$(date +%Y%m%d)/
```
- [ ] Snapshot completed and size looks reasonable

### 2.4 Snapshot Kasm
```
sudo rsync -av --delete /opt/kasm/ /home_backup/kasm-snapshot-$(date +%Y%m%d)/
```
- [ ] Kasm snapshot completed

---

## Phase 3 — The Upgrade

### 3.1 Install the Upgrade Plugin
```
sudo dnf install dnf-plugin-system-upgrade
```
- [ ] Plugin installed

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
- [ ] Confirm OS version: `cat /etc/fedora-release`
- [ ] Confirm kernel: `uname -r` (expect 6.17.x)
- [ ] Confirm desktop (KDE Plasma) is working
- [ ] Note: keep older kernel entries in GRUB as fallback if kernel 6.17.8+ has issues

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
```
for d in /opt/containers/*/; do
  name=$(basename "$d")
  sudo systemctl restart "$name" 2>/dev/null || true
done
```
Or restart individually as needed.

- [ ] All containers started without errors: `docker ps`
- [ ] Check for any exited containers: `docker ps --filter status=exited`

### 5.2 Per-Container Checks

| Container | Check | Status |
|-----------|-------|--------|
| actualbudget | UI loads on port 5007 | |
| convertx | UI accessible | |
| excalidraw | UI loads on port 5000 | |
| filestash | UI accessible | |
| garage | `garage status` returns healthy | |
| gitea | UI loads; SSH on port 2223 works | |
| glean | UI accessible | |
| home_file_server | UI accessible | |
| karakeep | UI loads on port 3000 | |
| kroki | Renders a test diagram | |
| n8n | UI loads on port 21712 | |
| openbao | `bao status` or API responds; **must unseal with 3 keys after restart** | |
| pastebooks | UI loads on port 8080 | |
| rsshub | Returns feed data | |
| wikijs | UI loads on port 3001 | |
| woodpecker-ci | Server + agent both healthy | |
| youtrack | UI loads on port 9000 | |

### 5.3 SELinux Labels — Containers Using `label:disable`
These containers use `security_opt: label:disable` and need revalidation after the SELinux policy update:
- [ ] **karakeep** — confirm still starts and functions (meilisearch + chrome inside)
- [ ] **n8n** — confirm still starts and workflow execution works
- [ ] **openbao** — confirm still starts and data directory is accessible
- [ ] **woodpecker-agent** — confirm agent connects to server and can reach Docker socket
- [ ] Check audit log for new denials: `sudo ausearch -m avc -c docker -ts today`

### 5.4 Backup Timers — Confirm Still Active
```
systemctl list-timers --all | grep backup
```
- [ ] All backup timers are loaded and scheduled

---

## Phase 6 — Kasm Upgrade (Separate Process)

Kasm Workspaces 1.18.1 lives at `/opt/kasm` and is NOT managed by dnf or docker-compose.

- [ ] Check Kasm F43 compatibility: https://www.kasmweb.com/docs/latest/release_notes.html
- [ ] Check for newer Kasm version compatible with F43 (current: 1.18.1)
- [ ] Follow Kasm's official upgrade procedure (not covered here — see Kasm docs)
- [ ] After Kasm upgrade: verify all Kasm services are healthy:
  ```
  sudo /opt/kasm/bin/utils/usage_stats.py
  docker ps | grep kasm
  ```

---

---

## Potential Issues — Published Reports (researched 2026-04-26)

> **Blocker assessment (2026-04-26):** None of the issues below are upgrade blockers. All have either a shipped fix or a confirmed workaround. See individual entries for rationale.

### ~~BLOCKER~~ CRITICAL — Docker CE + kernel 6.17 iptables — **NOT A BLOCKER: fix shipped in Docker CE v29.x**

**Risk level: HIGH for this system** (17 Docker Compose services depend on Docker Engine)

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

### ~~BLOCKER~~ HIGH — AMD amdgpu page faults and crashes (kernel 6.17.9+) — **NOT A BLOCKER: you are already running the fixed kernel**

**Risk level: HIGH for this system** (AMD Radeon PRO WX 7100, amdgpu driver)

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

### WARNING — Salt / SaltStack minion broken on Fedora 43 (Python 3.14) — **confirmed 2026-05-16**

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

**This host (kevin.kevininscoe.com, Fedora 42, Python 3.13):** Salt master is unaffected. Do NOT upgrade this host to Fedora 43 until Salt Python 3.14 compat is confirmed working.

---

### LOW — Snap / squashfs on F43

Snapd 2.72 is packaged for F43 and generally works. One known failure mode: snap apps fail to mount if `fuse` / `squashfuse` packages are missing.

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

### Kernel boot failure (6.17.8+ issue)
- At GRUB, select an older kernel to boot
- After stable boot: `sudo dnf install kernel-<older-version>` to pin if needed

### General dependency conflicts
```
sudo dnf system-upgrade download --releasever=43 --allowerasing
```
Review the package list carefully before confirming.

---

## After Upgrade QA

A systematic QA pass to run after the system is back up and containers are restarted. Steps marked **[AI]** can be run by an AI agent via shell commands. Steps marked **[HUMAN]** require visual/interactive verification.

### QA-1 — OS and Kernel

**[AI]**
```bash
cat /etc/fedora-release                          # expect: Fedora release 43
uname -r                                         # expect: 6.17.x-NNN.fc43.x86_64
rpm -q kernel | sort                             # confirm multiple kernels retained
rpm --version                                    # expect: RPM version 6.x
dnf --version                                    # expect: 5.x
getenforce                                       # expect: Enforcing
systemctl --failed --no-pager                    # expect: 0 units failed
```

- [ ] OS is Fedora 43
- [ ] Kernel is 6.17.x (note exact version; if 6.17.9+ and GPU issues occur, fall back)
- [ ] At least 2 kernel entries available in GRUB
- [ ] RPM 6.x confirmed
- [ ] DNF 5.x confirmed
- [ ] SELinux Enforcing
- [ ] No failed systemd units

### QA-2 — Core System Services

**[AI]**
```bash
sudo systemctl status httpd --no-pager           # Apache reverse proxy
sudo systemctl status tailscaled --no-pager      # Tailscale
sudo systemctl status certbot-renew.timer --no-pager
sudo systemctl status snapd --no-pager
sudo systemctl status docker --no-pager
docker --version                                 # must show 29.x
```

- [ ] httpd running
- [ ] Tailscale connected: `tailscale status`
- [ ] certbot-renew.timer active and scheduled
- [ ] snapd running
- [ ] Docker Engine running and v29.x

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

- [ ] All expected containers show `Up` — none in `Exited` or `Restarting`
- [ ] Containers using `label:disable` are running: karakeep, n8n, openbao, woodpecker-agent

### QA-5 — Container Health Endpoints

**[AI]** — run each curl and check for expected response:

```bash
# rsshub
curl -sf https://rsshub.kevininscoe.com/healthz && echo OK

# woodpecker-ci
curl -sf http://127.0.0.1:28706/healthz && echo OK

# openbao — must show "sealed":false
curl -s https://openbao.kevininscoe.com/v1/sys/health | python3 -m json.tool

# garage
sudo docker exec garage garage status

# gitea
curl -sf https://git.kevininscoe.com/api/v1/version && echo OK

# wikijs (allow 60s after restart)
curl -sf -o /dev/null -w "%{http_code}" https://wikijs.kevininscoe.com

# pastebooks
curl -sf -o /dev/null -w "%{http_code}" http://127.0.0.1:8080

# actualbudget
curl -sf -o /dev/null -w "%{http_code}" http://127.0.0.1:5007

# n8n
curl -sf -o /dev/null -w "%{http_code}" http://127.0.0.1:21712

# excalidraw
curl -sf -o /dev/null -w "%{http_code}" http://127.0.0.1:5000

# karakeep
curl -sf -o /dev/null -w "%{http_code}" http://127.0.0.1:3000

# youtrack — expect 200 after migration completes (60-90s)
curl -sf -o /dev/null -w "%{http_code}" http://127.0.0.1:9000
```

| Container | Expected | Pass? |
|-----------|----------|-------|
| rsshub | `ok` (200) | |
| woodpecker-ci | `{"status":"ok"}` (200) | |
| openbao | `"sealed": false` | |
| garage | healthy output | |
| gitea | JSON with version | |
| wikijs | 200 | |
| pastebooks | 200 | |
| actualbudget | 200 | |
| n8n | 200 | |
| excalidraw | 200 | |
| karakeep | 200 | |
| youtrack | 200 | |

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

All backup.sh files must show `bin_t` in their label. If any show `container_file_t`:

```bash
sudo chcon -t bin_t /opt/containers/<name>/backup.sh
```

- [ ] All backup.sh files have `bin_t` label

### QA-9 — Backup Timers Active

**[AI]**
```bash
systemctl list-timers --all | grep -E "backup|purge"
```

- [ ] All 14 backup/purge timers listed and showing a next trigger time
- [ ] No timer in `failed` state

Run one manual backup to confirm the full pipeline works end-to-end:
```bash
sudo systemctl start woodpecker-ci-backup.service
journalctl -u woodpecker-ci-backup.service -n 20 --no-pager
```

- [ ] Manual backup completes without error

### QA-10 — Kasm Workspaces

**[AI]**
```bash
sudo docker ps | grep kasm
sudo /opt/kasm/bin/utils/usage_stats.py 2>/dev/null | head -20
```

- [ ] All 8 Kasm containers are `Up`
- [ ] `kasm.service` is active: `systemctl status kasm`

**[HUMAN]**
- [ ] Log into Kasm web UI and launch a workspace session to verify end-to-end function
- [ ] kasm-network-plugin.service running: `systemctl status kasm-network-plugin`

### QA-11 — Snap Apps

**[AI]**
```bash
snap list
snap refresh --list 2>/dev/null || echo "up to date"
systemctl status snapd --no-pager
```

- [ ] All previously installed snaps listed: obs-studio, shortwave, ffmpeg-2404, core20/22/24, gnome-42/46-2204/2404
- [ ] snapd running without errors

**[HUMAN]**
- [ ] Launch one snap app (e.g. shortwave) to confirm it opens

### QA-12 — AMD GPU Stability

**[HUMAN]** — monitor for 30 minutes of normal use after upgrade:

```bash
# Check for GPU errors in kernel log
sudo journalctl -k | grep -i "amdgpu\|gpu.*fault\|drm.*error" | tail -20
```

- [ ] No `GCVM_L2_PROTECTION_FAULT_STATUS` errors in kernel log
- [ ] Desktop rendering looks correct (no artifacts)
- [ ] If instability occurs: note kernel version (`uname -r`) and fall back to previous kernel in GRUB

### QA Sign-off

- [ ] All QA-1 through QA-12 checks passed (or failures documented with workarounds applied)
- [ ] Date/time of completed QA: _______________
- [ ] Kernel version running: _______________
- [ ] Any open issues: _______________

---

## Post-Upgrade Notes

- [ ] Update this checklist with any issues encountered and how they were resolved
- [x] Create RUNBOOK.md files for the 7 containers that were missing them — DONE 2026-04-26 (actualbudget, excalidraw, karakeep, n8n, pastebooks, wikijs, youtrack)
- [x] Add Fedora 43 upgrade notes to existing RUNBOOK.md files — DONE 2026-04-26 (convertx, filestash, garage, gitea, glean, home_file_server, kroki, openbao, rsshub, woodpecker-ci)
- [ ] Run `sudo dnf distro-sync` to realign any 3rd-party packages to F43 equivalents
- [ ] **Bump Fedora version references from 42 → 43** now that the FLDW is actually on F43. This was deliberately deferred during the 2026-07-22 `fedora/` → `FLDW/` rename because the host was still on F42, and `~/ai/me.md` must reflect real state:
  - `~/ai/me.md` — change "Fedora Linux 42" → "Fedora Linux 43" in the *Systems and Environments* section (this is the one file confirmed correct at F42 today; it needs the bump post-upgrade).
  - Confirm `~/ai/directives/kevins-federated-unix-universe.md` and the service catalog (`~/Projects/private/fedora-dashboard/kevins-federated-unix-universe-services.md`) still read "Fedora Linux 43" — both were set ahead of the upgrade, so post-upgrade they simply become accurate (no edit expected, just verify).
