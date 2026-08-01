# TODO

Human-owned task list for this host. Delete this file once every item is checked off.

## Research: kernel 7.x on the Radeon PRO WX 7100 (GCN4)

**Raised:** 2026-08-01, during the Fedora 42 → 43 upgrade prep.
**Status:** open — this is the largest unresearched risk carried into F43.

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

### Tasks

- [ ] Search for GCN4 / Polaris regression reports against kernel 7.x — the canonical place is
      [drm/amd GitLab issues](https://gitlab.freedesktop.org/drm/amd/-/issues); also check the
      Fedora Discussion forum and the `amd-gfx` mailing list
- [ ] Confirm whether GCN4 / Polaris is still fully supported in 7.x, or has been moved toward
      legacy handling
- [ ] Record the running kernel version and any GPU log entries after the upgrade
      (`uname -r`; `sudo journalctl -k | grep -i 'amdgpu\|gpu.*fault\|drm.*error'`)
- [ ] Watch for 30+ minutes of real use, including something GPU-intensive, per QA-12 in
      `fedora-42-to-43-upgrade/FEDORA-UPGRADE-43.md`
- [ ] Write the finding back into the upgrade checklist so the next upgrade inherits the answer
      rather than the question

---

> **Glean's decommission moved out of this file on 2026-08-01.** It now lives in
> `/opt/containers/TODO.md`, with the containers repo that manages it. Glean was stopped and
> disabled before the F43 upgrade; its data is intact, and the open question is whether the stored
> articles need keeping.
