# TODO

Human-owned task list for this host. Delete this file once every item is checked off.

## FIRST — restore `setroubleshootd` (a safety net is currently switched off)

**Raised:** 2026-08-03. **Priority: do this before anything else in this file.**

- [ ] **Unmask and restart `setroubleshootd`.**
      ```bash
      sudo systemctl unmask setroubleshootd
      sudo systemctl start setroubleshootd
      systemctl is-active setroubleshootd    # expect: active
      ```

**Why it is off.** A failed attempt to convert `~/bin/backup.sh` from cron to systemd on
2026-08-03 put `rsync` into the confined `rsync_t` SELinux domain, generating ~3,700 AVC denials
in a few minutes. `setroubleshootd` raised a KDE desktop notification for each one and made the
desktop unusable, so it was stopped and masked at Kevin's instruction to restore usability.

**Why this must not stay this way.** Masking `setroubleshootd` disables the *only* channel that
puts an unexpected SELinux denial in front of a human on this workstation. Auditing itself is
untouched — `sudo ausearch -m avc -ts today` and `sudo sealert -a /var/log/audit/audit.log` still
work — but nothing will *tell* anyone; it has to be gone looking for. This is a temporarily
disabled safety net of exactly the kind that quietly becomes permanent, which is why it is the
first item in this file rather than a footnote to the backup work.

**Do it once the `/home_backup` and `/root_backup` label repair is settled** — until then a
relabel pass could itself produce denials and re-flood the desktop.

Verified while masking: the mask genuinely holds. `org.fedoraproject.Setroubleshootd.service`
delegates through `SystemdService=setroubleshootd.service` with `Exec=/bin/false`, so D-Bus
activation defers to systemd and cannot start it behind the mask.

## Research: kernel 7.x on the Radeon PRO WX 7100 (GCN4)

**Raised:** 2026-08-01, during the Fedora 42 → 43 upgrade prep.
**Status:** **largely answered 2026-08-02 by direct evidence — the card is healthy on kernel
7.1.5.** A 35-minute instrumented GPU soak found zero faults, ring timeouts, resets, or DRM
errors. What remains open is the literature search (has anyone else reported GCN4 regressions on
7.x?) and one untested subsystem — the video decode ring. See "What the soak found" below.

### What the concern is, plainly

The kernel is the part of Linux that drives the hardware, including the `amdgpu` driver for the
graphics card. This upgrade moves the kernel **from 6.19.14 straight to 7.1.x in a single step**,
because Fedora 43's updates repo ships kernel 7.1.5.

A normal Fedora upgrade nudges the kernel a little — 6.19 to 6.20. This one crosses from the 6.x
series into 7.x. The major-number bump does not by itself mean "breaking change" (Linus rolls the
number over when the minor number gets large), but **a jump this wide lands many releases' worth of
driver changes at once** instead of letting them arrive gradually.

The specific worry is the card's age. The **Radeon PRO WX 7100** is a 2016 part on AMD's **GCN4
(Polaris)** architecture. Old GPUs get the least testing — AMD's attention is on current hardware,
GCN-era support occasionally regresses, and cards are eventually moved to "legacy" status where
they stop receiving fixes.

**"Unresearched" means the evidence is absent, not bad.** No reports were found of the WX 7100
working well on 7.1, and none were found of it breaking. Nobody appears to have written it down.

### What failure would look like

- Screen artifacts, or the desktop freezing / the compositor restarting
- GPU hangs or resets in `journalctl -k`
- Everything becoming very slow because rendering fell back to software

### The safety net already in place

Kernel `6.19.14-108.fc42` stays installed and stays selectable in GRUB. If 7.1.x misbehaves,
reboot, pick the old kernel, and the machine is back to its pre-upgrade behaviour. Do not let
`dnf autoremove` or a low `installonly_limit` reap the F42 kernels until 7.1.x has proven itself.

### What the soak found — 2026-08-02

**The worry did not materialise.** Kernel `7.1.5-101.fc43` drives this 2016 GCN4 card without
complaint. Thirty-five minutes of sustained load produced **zero** GPU faults, ring timeouts,
resets, or DRM errors, and none of the three failure signatures listed above appeared: no
artifacts, no compositor restart, and no fallback to software rendering — OpenGL 4.6 with the
`radeonsi` driver and ACO shader compiler throughout.

The card ran hot but correctly: peak 93 °C against a 99 °C critical point, and peak 95.0 W against
its 95 W cap. It throttled on **power**, not temperature, which is the healthy signature.

**On the "moved to legacy" fear specifically — it has not been.** Polaris/GCN4 is still fully
supported by `amdgpu` in kernel 7.x and by Mesa 25.3.6. All 9 IP blocks initialized at boot,
including both video engines (`uvd_v6_3_0`, `vce_v3_4_0`), each reporting its firmware version.
The one real GCN4 limitation is **ROCm** (GPU compute), which dropped `gfx803` years ago — that is
unrelated to graphics or video, and long predates F43.

### Tasks

- [ ] Search for GCN4 / Polaris regression reports against kernel 7.x — the canonical place is
      [drm/amd GitLab issues](https://gitlab.freedesktop.org/drm/amd/-/issues); also check the
      Fedora Discussion forum and the `amd-gfx` mailing list.
      **Lower priority now** — this host has its own direct evidence; the search would only
      confirm it or surface a latent issue not yet hit.
- [x] Confirm whether GCN4 / Polaris is still fully supported in 7.x, or has been moved toward
      legacy handling — 2026-08-02. **Fully supported, not legacy.** See above.
- [x] Record the running kernel version and any GPU log entries after the upgrade — 2026-08-02.
      Kernel `7.1.5-101.fc43.x86_64`; GPU log entries clean across boot and soak.
- [x] Watch for 30+ minutes of real use, including something GPU-intensive, per QA-12 in
      `fedora-42-to-43-upgrade/FEDORA-UPGRADE-43.md` — 2026-08-02, 35 min instrumented soak.
      **Caveat:** gfx ring only. The video (UVD) ring could not be tested — see below.
- [x] Write the finding back into the upgrade checklist so the next upgrade inherits the answer
      rather than the question — 2026-08-02. Written into QA-12 of the F43 checklist and into the
      F44 planning notes' prerequisites.

### Closed by the second soak — 2026-08-03

- [x] **Hardware video decode restored and the UVD/VCE rings proven.**
      `mesa-va-drivers-freeworld` `25.3.6-1.fc43` installed from RPM Fusion, **both arches**
      (the `i686` one matters — Steam, Wine and Bottles use the 32-bit graphics stack). VAAPI
      profiles went **3 → 15**; H.264 and HEVC decode *and* encode all came back.
      A 15-minute soak driving 4K HEVC Main 10 decode and H.264 encode simultaneously found
      **zero** faults, zero software-decode fallbacks, and zero encode errors.
      This was a packaging gap, not a hardware or card-age problem — stock Fedora strips these
      codecs for patent reasons on every GPU, current models included.
      Logged in `~/ai/fedora/CHANGELOG.md`; full detail in QA-12 of the F43 checklist.

---

## Three failed units — surfaced by F43 QA-1, but not upgrade damage

**Raised:** 2026-08-03, during the F43 post-upgrade QA pass.
**Status:** open — each diagnosed, none caused by the upgrade.

All three failed on scheduled runs on 2026-08-03, two days after the upgrade, and each was
traced to its own unrelated cause. They are listed here so they are not lost, and so a future
reader does not mistake them for F43 fallout. Full context in QA-1 of
`fedora-42-to-43-upgrade/FEDORA-UPGRADE-43.md`.

- [ ] **`pcm-nightly-ingest-commit.service` (user unit) — git merge conflict needing triage.**
      The failing host unit `check-pcm-nightly-ingest` is only the *monitor*, and it is working
      correctly. The ingest job aborted because a standalone resync of `origin/main` into the
      `ingest` branch conflicts outside the generated maps, in `moc/docker-containers.md` and
      `notes/qa76-docker-sbom-command-...md`. Needs a human to resolve the conflict.
      *This is not a regression of the 2026-08-01 fix* — that fix addressed a different fault.
- [ ] **`youtrack-export.service` — bug in the exporter.**
      `TypeError: 'int' object is not iterable` at
      `~/Projects/private/youtrack.kevininscoe.com/scripts/export-all-youtrack-data/export.py:622`,
      in `_names`, where `for v in seq or []` receives an `int`. A YouTrack API field is coming
      back as a scalar where the script expects a sequence. The script **failed safe**: retention
      was skipped, so the previous good export survives. Fix is a guard in `_names`.
- [x] **Stale Glean cron entry failing silently every night — REMOVED 2026-08-03.**
      Found by QA-9. `/etc/cron.d/glean-backup` ran `/opt/containers/glean/backup-glean.sh` as
      root at 02:30 daily, but Glean was decommissioned on 2026-08-01 and its containers removed,
      so every run failed with `Error response from daemon: No such container: glean-postgres`.
      **Nothing alerted on it** — cron is silent on non-zero exit, no monitor watched the log, and
      with no systemd unit it fell outside `systemctl --failed` and `check-all-backups.sh` alike.
      Backed up to `/home/backups/removed-cron-entries/glean-backup.removed-20260803`, then
      deleted. No `glean` references remain in `/etc/cron.d`, `/etc/crontab`, `/etc/cron.daily` or
      `/etc/logrotate.d`; `crond` still active. `/opt/containers/glean/` and its script were left
      in place — the stored-articles question stays in `/opt/containers/TODO.md`.
      Logged: `~/ai` `dc0dcab`, pushed.

- [ ] **Sweep for other cron-only jobs with no monitoring.** Raised 2026-08-03 by the item above.
      The Glean entry failed nightly for two days unnoticed because a `/etc/cron.d` job has no
      systemd unit, so it is invisible to `systemctl --failed`, to `~/admin/check-all-backups.sh`,
      and to the Telegram alert paths — the three places a failure would normally surface.
      Any other cron-only job on this host has the same blind spot. Enumerate `/etc/cron.d`,
      `/etc/crontab`, `/etc/cron.*`, and user crontabs, and decide per job whether to convert it to
      a systemd unit + timer (which `when-establishing-monitoring-for-a-job-or-service.md` would
      then cover) or to give it explicit alerting.
- [ ] **`recollindex-overnight.service` — timed out, 39.4 GB memory peak.**
      Ran 01:00 → 05:03, consumed 30 min CPU, hit a **39.4 GB** memory peak, then was SIGKILLed
      when the unit's timeout expired. Decide whether to raise the timeout, cap the indexer's
      memory, or narrow what it indexes — a 39 GB peak is worth understanding before simply
      granting it more time.

---

> **Glean's decommission moved out of this file on 2026-08-01.** It now lives in
> `/opt/containers/TODO.md`, with the containers repo that manages it. Glean was stopped and
> disabled before the F43 upgrade; its data is intact, and the open question is whether the stored
> articles need keeping.
