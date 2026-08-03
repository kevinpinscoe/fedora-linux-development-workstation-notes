# TODO

Human-owned task list for this host. Delete this file once every item is checked off.

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

> **Glean's decommission moved out of this file on 2026-08-01.** It now lives in
> `/opt/containers/TODO.md`, with the containers repo that manages it. Glean was stopped and
> disabled before the F43 upgrade; its data is intact, and the open question is whether the stored
> articles need keeping.
