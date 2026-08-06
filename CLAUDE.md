# CLAUDE.md

Guidance for Claude Code (claude.ai/code) when working in this repository.

## What this repo is

A public knowledge base of **Fedora Linux findings** — what actually happened during real version upgrades on a working KDE Plasma desktop, and the issues, quirks, and fixes that came out of them.

There is no build system, no test suite, and no application code. Everything here is Markdown.

The value of these notes is **specificity**. They record real upgrades on real hardware: the exact error message, what it turned out to mean, what fixed it, and the command that proves the fix worked. General Fedora advice is already everywhere; observed outcomes with their diagnosis are not.

## Scope — what belongs here

The test is **usefulness to someone else running Fedora**: would a stranger learn anything from this?

| Belongs here | Does not |
|---|---|
| A Fedora bug, regression, or packaging gap, and how it presented | One machine's service inventory, port map, or container list |
| An upgrade failure mode with its root cause and tell-tale symptom | Which of *this* owner's services were down on a given day |
| A behavior that changed between Fedora releases | Personal configuration choices with no general lesson |
| Hardware/driver findings tied to an identifiable chipset or kernel version | Host-specific paths, hostnames, credentials, or internal addresses |
| A checklist or method others could reuse | Task tracking for one person's machine |

Host-specific operational detail is maintained separately, in the `fedora-linux-development-workstation-notes` repo, and is deliberately kept out of this one. When a finding has both a general and a host-specific half, **publish the general half here and leave the specifics out** — usually that means describing the symptom, the cause, and the fix, while replacing an inventory with a command that regenerates it.

A useful heuristic: if removing the machine's identity would make the note meaningless, it probably belongs in the private repo. If the note survives that removal intact, it belongs here.

## Provenance — the machine these findings come from

Recorded so a reader can judge whether a finding applies to them. Several notes here are hardware- or stack-specific and will not generalize.

- **OS:** Fedora Linux (notes span 42 → 43), **SELinux enforcing**, targeted policy
- **Desktop:** KDE Plasma 6 on Wayland
- **GPU:** AMD Radeon PRO WX 7100 — GCN4 / Polaris, 2016 part, `amdgpu` open-source driver only, no proprietary stack
- **Packaging:** `dnf` (DNF 5 from F43 onward), `snap`, RPM Fusion, Flatpak
- **Containers:** Docker via Fedora's `moby-engine` package — **not** Docker CE, which matters whenever a fix names `docker-ce` packages
- **Kernel:** the F42 → F43 notes cover a 6.19 → 7.1 jump in a single step

SELinux being **enforcing** is the single most load-bearing item in that list. A large share of these findings would not reproduce on a permissive or disabled host, and several are entirely about label and domain transitions.

## Conventions

- One subdirectory per topic or major event — e.g. `fedora-42-to-43-upgrade/`.
- Files prefixed `notes-` are research and planning notes: conversational, and possibly incomplete or superseded.
- Files without that prefix are actionable references — checklists, runbooks, inventories.
- Checklist items use `- [ ]` / `- [x]` GitHub-flavored task lists. When closing one, mark it `[x]` and add the resolution date and a short description **inline**, so the file reads as a record rather than a to-do.
- Corrections are made **in place**. If a note turns out to be wrong, fix it and say so where it stood — do not add a separate "fixed" or "errata" file. A checklist that quietly contradicts itself is worse than one that admits it was wrong.
- `resume.sh` is gitignored and must never be committed.

## Redaction

This repository is **public**, and some of its source material is not.

`.gitignore` carries `**/*.local.md` and `**/*.local.txt`. Host detail that would be reconnaissance material — port maps, lists of services running with SELinux confinement disabled, and full `rpm -qa` package manifests — stays in those files and is never published. A package-and-version inventory of an internet-reachable host is the strongest such artifact.

Where a published note needs that data, it carries a **command that regenerates it live** instead of the data itself. For example, rather than pasting a port table:

```bash
docker ps --format '{{.Names}}\t{{.Ports}}'
```

This keeps the analysis useful without publishing the inventory it was derived from.

## Adding new content

- **New upgrade or migration:** create `fedora-<from>-to-<to>-upgrade/`, or `topic-<name>/` for anything else.
- **Before an upgrade, capture a baseline** — and include `rpm -qa` in it, kept as `*.local.txt`. A baseline that records running services but not installed packages cannot answer "was this already installed before?", which is the question that actually comes up afterward.
- **Resolved issues:** update the relevant checklist in place, with the date.
- **Record what was ruled out, not just what was found.** The reasoning that eliminated a wrong hypothesis is often the most reusable part of a note, and it is what stops the next person re-investigating a dead end.
