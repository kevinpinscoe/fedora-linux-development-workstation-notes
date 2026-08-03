# Fedora 43 → 44 upgrade — planning notes

**Status:** research only. Nothing is scheduled and no work has begun.
**Opened:** 2026-08-01
**Blocked on:** the Fedora 43 post-upgrade QA pass is not finished. See "Prerequisites" below.

These are planning notes, not a checklist. The actionable checklist —
`FEDORA-UPGRADE-44.md`, modelled on `../fedora-42-to-43-upgrade/FEDORA-UPGRADE-43.md` — gets
written once the open questions below are answered and a date is chosen. Writing it now would
mean inventing content for an upgrade nobody has researched yet.

---

## The headline: this is a much smaller jump than F42 → F43

Every number below was read off the live host or off Fedora's own metadata on **2026-08-01**, not
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

**Researched 2026-08-01 — see [`notes-mariadb-11-migration.md`](notes-mariadb-11-migration.md)
for the full findings.** The short version, which downgrades this risk considerably:

- **In-place upgrade is supported** and major versions may be skipped — 10.11 → 11.8 direct is a
  sanctioned path. A dump/restore is not required.
- **The dataset is tiny** — two MediaWiki databases totalling ~360 MB. `backupmysql` already
  writes a 41 MB gzipped full dump six times a day.
- **Only two non-InnoDB tables**, both MediaWiki `searchindex` (MyISAM), and both regenerable
  with `rebuildtextindex.php`. Not authoritative data.
- **Only affects the host instance.** `pastebooks-db` (`mysql:8.4`) and three Postgres containers
  pin their own images and are untouched by a Fedora release upgrade.
- **Downgrade is not supported** — "MariaDB server is not designed for downgrading." The rollback
  is a restore from dump, taken immediately before and *verified restorable*. This is the single
  most important prerequisite.
- **The change most likely to bite** is `innodb_snapshot_isolation` defaulting to `ON` in 11.8,
  which makes Repeatable Read conflict detection stricter and can raise `ERROR 1020` on
  concurrent `UPDATE`/`DELETE`. MediaWiki does concurrent writes. Mitigation is a one-line
  `my.cnf` revert.
- **11.8 is LTS**, maintained to June 2028 — this is LTS → LTS, which weakens the case for
  skipping F44 purely to avoid the migration.

Recommended sequencing: rehearse the migration in a throwaway 11.8 container against a restored
dump *before* the OS upgrade, so the database work and the distro work are never being debugged
at the same time.

---

## Ecosystem readiness — checked 2026-08-01, all green

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

- [x] **Reboot and confirm the backup-timer burst is gone** — **DONE 2026-08-02.** The 15:40 boot
      produced zero `*-backup.service` starts; the timers armed normally and fired no jobs. The
      fix is now verified behaviourally, not just structurally.
- [x] **GPU soak (QA-12)** — **DONE 2026-08-02, and it is the F44 kernel answer.** 35 minutes of
      sustained `glmark2` load: **zero** GPU faults, ring timeouts, resets, or DRM errors. Peak
      93 °C and 95.0 W against a 95 W cap, throttling power-limited rather than thermal. **GCN4 /
      Polaris is healthy on kernel 7.1.5**, so the "old card on a new kernel" risk does not carry
      into F44 — F44's kernel line is near-identical to F43's. Full results in QA-12 of
      `../fedora-42-to-43-upgrade/FEDORA-UPGRADE-43.md`.
      The **video engines were proven separately on 2026-08-03** — see the next item.
- [x] **`mesa-va-drivers-freeworld` — installed 2026-08-03, both arches.** Stock Fedora's
      `mesa-va-drivers` strips H.264/HEVC for patent reasons, so all such video had been decoding
      on the CPU while the card's initialized UVD block sat idle. VAAPI profiles went **3 → 15**,
      restoring H.264 and HEVC decode *and* encode. A 15-minute soak of simultaneous 4K HEVC
      decode and H.264 encode found **zero** faults. Not upgrade damage as far as could be shown.
      **Carry into F44:** `mesa-va-drivers-freeworld` is an RPM Fusion package, so it is exactly
      the class most likely to be dropped if the repo has not built for the new release when the
      upgrade runs. Check it explicitly in the post-upgrade package diff.
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

## Baseline gap to close before F44 — capture a package manifest

**Raised 2026-08-02.** The F43 pre-upgrade baseline was thorough about *state* — services,
containers, published ports, 38 backup timers, third-party repo readiness, snaps — but it never
captured the **installed package list**. That gap surfaced immediately during F43 QA: on finding
`mesa-va-drivers-freeworld` missing, there was no way to answer *"was it installed before the
upgrade?"*, and `dnf history` did not settle it either.

Being unable to answer that question is worse than the missing package itself. Any post-upgrade
"is this thing missing because the upgrade dropped it?" is unanswerable without a before-picture,
and third-party packages (RPM Fusion, COPR) are exactly the ones most likely to be dropped when a
repo has not yet built for the new release.

**Capture two manifests, not one.** They answer different questions:

```bash
# Full NEVRA — answers "what version was I on?"
rpm -qa | sort > installed-packages-before-fedora-44-<DATE>.local.txt

# Names only — answers "what disappeared?"
rpm -qa --qf '%{NAME}\n' | sort -u > installed-package-names-before-fedora-44-<DATE>.local.txt
```

**Why two.** A dropped package cannot be found by diffing NEVRA lines, because *every* line
changes when versions bump across a release. The comparison has to be name-against-name. And the
name cannot be recovered from a NEVRA string by splitting on hyphens — most package names contain
hyphens themselves, so `cut -d- -f1` turns `mesa-va-drivers-25.3.6-3.fc43.x86_64` into `mesa`.
Only `rpm` knows where the name ends, so ask it directly with `--qf '%{NAME}\n'`.

After the upgrade, the packages that vanished are then a clean one-liner:

```bash
rpm -qa --qf '%{NAME}\n' | sort -u > installed-package-names-after-fedora-44-<DATE>.local.txt
comm -23 installed-package-names-before-fedora-44-<DATE>.local.txt \
         installed-package-names-after-fedora-44-<DATE>.local.txt
```

Expect third-party packages (RPM Fusion, COPR) to dominate that list — they are the ones a major
upgrade drops when the repo has not yet built for the new release.

**Captured 2026-08-02**, ahead of any F44 work:

| File | Lines | Answers |
|---|---|---|
| `installed-packages-before-fedora-44-2026-08-02.local.txt` | 8,480 | what version was installed |
| `installed-package-names-before-fedora-44-2026-08-02.local.txt` | 8,236 | what was installed at all |

- [x] Add the `rpm -qa` capture to the F44 pre-upgrade baseline procedure — **done 2026-08-02**,
      both manifests captured and stored in this directory.
- [ ] Re-capture both immediately before the upgrade actually runs — the 2026-08-02 files are a
      useful early snapshot, but the authoritative before-picture is the one taken on the day.
- [ ] Run the post-upgrade `comm` diff as an explicit QA step, rather than discovering gaps by
      accident as happened with `mesa-va-drivers-freeworld` during F43 QA.
- [x] Keep the manifests out of git — **done.** They carry the `.local.txt` suffix and
      `.gitignore` now excludes `**/*.local.txt`. **This repo is public**, and an exact
      software-and-version inventory of an internet-reachable host is the strongest
      reconnaissance artifact it could contain.

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
