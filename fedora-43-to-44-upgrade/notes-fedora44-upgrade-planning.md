# Fedora 43 → 44 upgrade — planning notes

**Status:** research only. Nothing is scheduled and no work has begun.
**Opened:** 2026-08-02
**Blocked on:** the Fedora 43 post-upgrade QA pass is not finished. See "Prerequisites" below.

These are planning notes, not a checklist. The actionable checklist —
`FEDORA-UPGRADE-44.md`, modelled on `../fedora-42-to-43-upgrade/FEDORA-UPGRADE-43.md` — gets
written once the open questions below are answered and a date is chosen. Writing it now would
mean inventing content for an upgrade nobody has researched yet.

---

## The headline: this is a much smaller jump than F42 → F43

Every number below was read off the live host or off Fedora's own metadata on **2026-08-02**, not
recalled. Package versions in a released Fedora move, so re-run these checks before committing to
a date — the method is recorded beside each so they can be repeated.

| Component | Now (F43) | F44 | Verdict |
|---|---|---|---|
| **Kernel** | `7.1.5-101.fc43` | `7.1.5-201.fc44` | **Same upstream version** |
| **Python** | `3.14.6-1.fc43` | `3.14.6-1.fc44` | Identical |
| **podman** | `5.8.4-1.fc43` | `5.8.4-1.fc44` | Identical |
| **httpd** | `2.4.68-1.fc43` | `2.4.68-1.fc44` | Identical |
| **haproxy** | `3.0.25-1.fc43` | `3.0.25-1.fc44` | Identical |
| **mariadb-server** | `10.11.18-2.fc43` | `11.8.8-3.fc44` | **Major version jump — see below** |

Method:

```bash
sudo dnf --releasever=44 --repo=fedora --repo=updates --refresh \
  repoquery --qf '%{version}-%{release}\n' <package>
```

### Why the kernel line matters most

The single largest risk carried into F43 was the kernel going `6.19.14 → 7.1.x` in one step, on a
2016 **Radeon PRO WX 7100 (GCN4 / Polaris)** — the concern written up in the repo root `TODO.md`.

**That risk does not repeat here.** F44's updates repo currently ships the *same upstream kernel*
this host already runs, `7.1.5`, differing only in the Fedora build number (`-101.fc43` →
`-201.fc44`). F44's GA kernel was actually older (`6.19.10-300.fc44`); updates carried it forward
to 7.1.5, the same place F43 landed.

The practical consequence is worth stating plainly: **whatever Sunday's GPU soak concludes about
kernel 7.1.5 on the WX 7100 answers the question for F44 as well.** If 7.1.5 is good on F43 it is
the same kernel on F44. If it is bad, that is a kernel problem to solve before F44 is worth
discussing at all. Either way the F43 TODO research is not wasted effort — it is a prerequisite
that pays for both upgrades.

Caveat: F44 is a live release and its kernel will move on. Re-check at decision time.

---

## The one real risk: MariaDB 10.11 → 11.8

This is the standout item and the reason this upgrade is not simply routine.

- F43 ships **MariaDB 10.11.18**, a long-term-support series.
- F44 ships **MariaDB 11.8.8**, a different major series.

A major MariaDB jump is not a drop-in package swap. It typically needs a `mariadb-upgrade` run
against the data directory, and the on-disk format and some SQL/config behaviour can change
between series. On this host that matters more than usual because MariaDB is not a leaf package —
`backupmysql` runs against it on a six-times-daily schedule, and container stacks sit on top of it.

**Unresearched.** No compatibility work has been done. Before F44 is scheduled:

- Read the MariaDB 11.x upgrade notes for anything that breaks a 10.11 data directory.
- Establish what the rollback is. A package downgrade will **not** undo a data-directory upgrade —
  the real rollback is a restore from dump, so a verified dump has to exist immediately before.
- Decide whether to do the MariaDB migration as its own change *before* the OS upgrade, so the two
  are not entangled. Debugging a database migration and a distro upgrade at the same time is the
  situation to avoid.

Note that the F43 upgrade already exercised the "dump exists and is restorable" path — see the
gitea-backup and youtrack restore-drill machinery — so the tooling is there.

---

## Ecosystem readiness — checked 2026-08-02, all green

The third-party repos that historically block a Fedora upgrade all have F44 content already:

| Repo | F44 |
|---|---|
| `docker-ce` | HTTP 200 |
| RPM Fusion free / nonfree | HTTP 200 / 200 |
| copr `dejan/lazygit` | HTTP 200 |
| copr `ilyaz/LACT` | HTTP 200 |
| copr `jdxcode/mise` | HTTP 200 |
| copr `lihaohong/yazi` | HTTP 200 |
| copr `phracek/PyCharm` | HTTP 200 |

Method — request each repo's `repodata/repomd.xml` for `fedora-44-x86_64` and check the status
code. A 404 means that repo has no F44 build and will be left behind by the upgrade.

No enabled third-party repo outside `fedora`/`rpmfusion`/`copr` interpolates `$releasever`, so
those are version-independent and unaffected.

This is a materially better starting position than F43 had, where the `sftpgo` repo's metadata was
404ing and had to be disabled mid-flight.

---

## Prerequisites — none of this starts until these close

Inherited from the F43 upgrade, tracked in the repo root `CHECKPOINT.md`:

- [ ] **Reboot and confirm the backup-timer burst is gone.** Eleven timers were fixed on
      2026-08-02 (`Requires=` removed from `[Unit]`); the fix is verified structurally but not yet
      behaviourally. Kevin is doing this Sunday.
- [ ] **GPU soak (QA-12)** — 30+ minutes of real use including something GPU-intensive, then
      re-check `journalctl -k`. Also Sunday. As above, this doubles as the F44 kernel answer.
- [ ] **Finish the F43 post-upgrade QA pass** in `../fedora-42-to-43-upgrade/FEDORA-UPGRADE-43.md`.
- [ ] **Close out the root `TODO.md`** kernel/GCN4 research and write the finding back into the
      checklist, so this upgrade inherits an answer rather than the question.
- [ ] **Clear the seven non-kernel `fc42` stragglers** — `ghostty`,
      `claude-desktop-unofficial`, `webkit2gtk4.0`, `javascriptcoregtk4.0`, `peek`, `xl2tpd`,
      `zfs-fuse`. Packages that failed to move F42 → F43 will not improve by adding another
      release on top; each needs to be updated, replaced, or deliberately dropped.
- [ ] **`dnsmasq.service`** — failed since before the F43 upgrade, still failed. Fix or retire it
      rather than carrying it into a third release.

---

## Timing

There is no deadline pressure, and that is worth using.

- F44 has been GA since roughly **April 2026** (Fedora's `releases.json` lists 44-1.7 images; the
  F44 metalink metadata timestamp corresponds to April). It is a mature release, not a fresh one.
- Fedora supports a release until about **one month after the N+2 release**. For F43 that means
  roughly one month after F45 ships, which on the usual cadence lands in **late 2026**.
  **Confirm the exact date before relying on it** — it is a schedule, and schedules slip.

So the realistic options:

1. **Upgrade F43 → F44 in the autumn**, once F43 QA is closed and MariaDB is sorted.
2. **Wait for F45 and skip straight to it.** Fedora supports skipping one release
   (N → N+2), and F43 → F45 is a supported path. This trades one upgrade event for one larger one.

Option 2 deserves serious consideration given how quiet F44 looks for this host — near-identical
kernel, Python, podman, httpd and haproxy. The main argument against it is that skipping does not
avoid the MariaDB migration; it only defers it and probably makes it larger.

**Recommendation:** decide this after the F43 QA closes, not now. The MariaDB research is the
input that actually settles it, and it can be done independently of either upgrade.

---

## Open questions

1. What does the MariaDB 10.11 → 11.8 migration actually require on this host, and what is the
   verified rollback?
2. Should MariaDB be migrated as a standalone change ahead of the OS upgrade?
3. F44 or skip to F45? (See Timing.)
4. What are F44's own release-notes changes beyond package versions — installer, DNF, SELinux
   policy, KDE Plasma version? **Not yet researched.** The version comparison above is a package
   diff, not a release-notes review, and the two are not the same thing.
5. What is F43's exact EOL date?
6. Kasm Workspaces — its fate was left undecided in Phase 6 of the F43 checklist and its
   8 containers are still exited. Decide before F44 rather than carrying it a third time.

---

## Next artifact

`FEDORA-UPGRADE-44.md` — the actionable phase-by-phase checklist, modelled on the F43 one
(Pre-upgrade prep → Backups → The upgrade → Post-upgrade verification → Container verification →
QA). It gets written when questions 1–4 above have answers and a date is chosen.
