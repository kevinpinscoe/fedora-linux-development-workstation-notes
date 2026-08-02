# MariaDB 10.11 → 11.8 migration research

**Opened:** 2026-08-01
**Why:** Fedora 44 ships `mariadb-server 11.8.8`; this host runs `10.11.18` on F43. This is the
one component that genuinely changes across the F43 → F44 upgrade, and it gates the
"F44 or skip to F45" decision in `notes-fedora44-upgrade-planning.md`.

**Conclusion up front: this is a smaller job than it first looked.** The dataset is tiny, the
only non-InnoDB tables are rebuildable, both consumers connect over a Unix socket, and MariaDB
supports the in-place upgrade path. The real work is a config review and a verified dump, not a
migration project.

---

## What is actually on this host

Read from the live server on 2026-08-01.

| Database | Tables | Size | Engines |
|---|---|---|---|
| `mediawiki_private` | 59 | 327.0 MB | InnoDB, MyISAM |
| `mediawiki_work` | 58 | 31.1 MB | InnoDB, MyISAM |
| `mysql` (system) | 31 | 3.5 MB | Aria, CSV, InnoDB |

**Total real data: ~360 MB.** That is small enough that a full logical dump and restore is a
routine operation, not an outage plan. `backupmysql` already produces a 41 MB gzipped full dump
six times a day.

- **datadir:** `/var/lib/mysql/`, `innodb_file_per_table=1`
- **Non-default plugins loaded:** compression providers only — `bzip2`, `lz4`, `lzma`, `lzo`,
  `snappy`. No Galera, no Spider in use, no replication configured.
- **MyISAM tables — exactly two:** `mediawiki_private.searchindex` and
  `mediawiki_work.searchindex`. Both are MediaWiki's full-text search index and are
  **regenerable** with `php maintenance/rebuildtextindex.php`. They carry no authoritative data,
  so they are a non-issue for the migration — worst case, rebuild them afterwards.

### Consumers

Only two, both MediaWiki instances running under the host Apache:

| Instance | Config | `$wgDBserver` | Database |
|---|---|---|---|
| private wiki | `/var/www/html/pw/LocalSettings.php` | `localhost` | `mediawiki_private` |
| work wiki | `/var/www/html/work/LocalSettings.php` | `localhost` | `mediawiki_work` |

`$wgDBserver = "localhost"` means the **Unix socket**, not TCP — that matters, see the
`require_secure_transport` note below.

**Not affected by this upgrade:** `pastebooks-db` runs `mysql:8.4` as a container and pins its own
image, and three Postgres containers likewise. A Fedora release upgrade does not touch container
images. The F44 MariaDB jump affects the **host instance only**.

---

## What MariaDB says about the upgrade path

Per [Upgrading between major MariaDB versions](https://mariadb.com/kb/en/upgrading-between-major-mariadb-versions/):

- **In-place upgrade is supported** — a dump/restore is *not* required. MariaDB's own wording is
  that you should be able to upgrade from any earlier version to the latest "usually in a few
  seconds."
- **Skipping major versions is supported.** 10.11 → 11.8 directly is a sanctioned path; there is
  no need to step through 11.4.
- **`mariadb-upgrade`** updates the `mysql` system tables, runs collation checks and recreates
  affected indexes, and validates table compatibility.

### The rollback is the real constraint

> "MariaDB server is not designed for downgrading."

A package downgrade does **not** undo a data-directory upgrade. A supported downgrade requires
no `ALTER TABLE`/`CREATE TABLE` since the upgrade, a pre-upgrade dump of the `mysql` database,
`innodb_fast_shutdown=0` before shutdown, and manual restoration of the old `mysql` database.

**Practical consequence for this host:** the rollback plan is *restore from dump*, and the dump
must be taken immediately before the upgrade and **verified restorable**, not merely written.
At 41 MB gzipped this is cheap. This is the single most important prerequisite on this page.

---

## Behaviour changes that could bite

From [What is MariaDB 11.8](https://mariadb.com/kb/en/what-is-mariadb-118/) and
[Upgrading from 11.4 to 11.8](https://mariadb.com/docs/server/server-management/install-and-upgrade-mariadb/upgrading/mariadb-community-server-upgrade-paths/upgrading-from-mariadb-11-4-to-mariadb-11-8):

### 1. `innodb_snapshot_isolation` now defaults to `ON` (was `OFF`)

The highest-relevance change here. Under Repeatable Read, conflict detection is stricter:
`DELETE`/`UPDATE` statements can now raise **`ERROR 1020`** where they previously succeeded, when
a row changed since the transaction's snapshot.

MediaWiki does concurrent writes (page saves, link table updates, job queue). This is the change
most likely to produce a real, visible fault on this host. Mitigation if it bites: set
`innodb_snapshot_isolation=OFF` in `/etc/my.cnf.d/mariadb-server.cnf` to restore 10.11 behaviour.
Worth pre-staging that line as a known rollback lever.

### 2. Default character set `latin1` → `utf8mb4`, collation → `uca1400_ai_ci`

Defaults apply to **newly created** objects, so existing MediaWiki tables keep their current
charset and are not silently rewritten. The risk is mixed-charset drift later — any table created
after the upgrade (a MediaWiki schema update, a new extension) picks up the new default and may
not match its neighbours. MediaWiki is historically sensitive to charset/collation mismatches.

Action: record the current per-table charset before upgrading so any drift is detectable
afterwards, and consider setting the old defaults explicitly in `my.cnf` if a mismatch appears.

### 3. `require_secure_transport` — **not a problem here**

One secondary source claimed 11.8 defaults this to `1`. **That is unverified** — MariaDB's own
11.8 release notes do not list it among changed defaults, and the claim appears to come from
Enterprise documentation. Treat it as unconfirmed.

It does not matter either way on this host: per
[Securing connections for client and server](https://mariadb.com/kb/en/securing-connections-for-client-and-server/),
**Unix sockets are classified as a secure transport**, alongside SSL/TLS and named pipes. Both
MediaWiki instances use `localhost` (socket) and `backupmysql` uses
`/var/lib/mysql/mysql.sock`. Every consumer on this host is already "secure" by that definition,
so even if the default did flip, nothing here would be refused.

### 4. `TIMESTAMP` range extended to 2106

Was `2038-01-19`, now `2106-02-07` on 64-bit. Beneficial, no action.

### 5. Removed / renamed variables

The 11.4 → 11.8 guide names only `wsrep_load_data_splitting` (Galera, not used here). The wider
10.x → 11.x span removes and renames more, so the 13 files in `/etc/my.cnf.d/` need a read-through
against the 11.8 variable list before the upgrade — an unknown variable makes the server refuse to
start, which is a loud failure but an avoidable one.

### 6. Optimizer re-weighting

11.x reworked cost weighting for SSD-era storage. Query plans can change. With a 360 MB dataset
and two low-traffic wikis this is close to irrelevant here, but it is why "it got slower" is a
plausible post-upgrade report rather than a mystery.

### Not applicable

Vector datatypes, replication compatibility, and Galera changes — none configured on this host.

---

## Bonus: 11.8 is LTS

10.11 is an LTS series and so is **11.8, maintained until June 2028**. This is an LTS → LTS move,
not a jump onto a short-lived release. That materially weakens the argument for skipping F44 to
avoid the migration — the destination is a supported long-term series either way.

---

## Proposed approach

**Do the MariaDB migration as its own change, before the OS upgrade.** Debugging a database
migration and a distro upgrade simultaneously is the situation to avoid, and this one can be
rehearsed independently.

Draft sequence — not yet a checklist, and not yet agreed:

1. Record current state: per-table charset/collation, `SHOW VARIABLES`, the 13 `/etc/my.cnf.d/`
   files.
2. Review `/etc/my.cnf.d/` against the 11.8 variable list; note anything removed or renamed.
3. Take a full dump and **verify it restores** into a scratch instance. This is the rollback.
4. Rehearse: restore the dump into a throwaway MariaDB 11.8 container, run `mariadb-upgrade`,
   point a test MediaWiki at it, exercise page save / search / history.
5. Only then decide in-place-on-F44 versus dump-and-restore.
6. Rebuild `searchindex` on both wikis afterwards and confirm search works.

Step 4 is the one that turns this from a risk into a known quantity, and it costs nothing but
time — the whole dataset fits in a container.

---

## Open questions

1. Is the MariaDB migration done standalone on F43 first (Fedora only ships 10.11 on F43, so this
   would mean upstream packages or a container), or accepted as part of the F44 upgrade?
   **This is the key sequencing question and it is not yet answered.**
2. Does either MediaWiki version have its own constraint on MariaDB 11.8? Not yet checked —
   MediaWiki's release notes state supported database versions and that has not been read.
3. Should the two wikis be moved into containers with a pinned MariaDB image instead, decoupling
   them from the Fedora release cycle entirely? That would make this the last time this question
   comes up.

---

## Sources

- [Upgrading between major MariaDB versions](https://mariadb.com/kb/en/upgrading-between-major-mariadb-versions/)
- [What is MariaDB 11.8](https://mariadb.com/kb/en/what-is-mariadb-118/)
- [Upgrading from MariaDB 11.4 to MariaDB 11.8](https://mariadb.com/docs/server/server-management/install-and-upgrade-mariadb/upgrading/mariadb-community-server-upgrade-paths/upgrading-from-mariadb-11-4-to-mariadb-11-8)
- [Securing connections for client and server](https://mariadb.com/kb/en/securing-connections-for-client-and-server/)
