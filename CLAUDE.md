# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A personal knowledge base of notes, observations, and fixes for a Fedora Linux KDE Plasma 6 developer workstation (`kevin.kevininscoe.com`). There is no build system, test suite, or application code — all content is Markdown documentation.

## Conventions

- One subdirectory per topic or major event (e.g. `fedora-42-to-43-upgrade/`).
- Files prefixed `notes-` are research/planning notes (conversational, may be incomplete).
- Files without a `notes-` prefix are actionable references: checklists, runbooks, inventories.
- Checklist items use `- [ ]` / `- [x]` GitHub-flavored Markdown task lists. Mark items `[x]` with the resolution date and a short description inline when they are completed.
- `resume.sh` is gitignored — it holds a `claude --resume <session-id>` command for the owner's convenience and should never be committed.

## Scope — what belongs here and what does not

This repo is about **the planning and QA of Fedora upgrades themselves**. Keep it to that.

Post-upgrade QA routinely turns up host problems that have nothing to do with the upgrade — a misconfigured job, a cron entry failing silently, a service nobody is monitoring. Those findings are **raised in `~/admin/TODO.md`** (the `fedora-admin` repo) for the host-administration agent and Kevin to investigate. Kevin is the gate on any action taken about them. Do not open a tracking section for them here.

The distinction is cause, not subject matter:

| Finding | Goes to |
|---|---|
| Caused by, or a regression from, the upgrade | this repo — the relevant `QA-*` section, and `TODO.md` if it stays open |
| Pre-existing, or unrelated to the upgrade, merely *found* during QA | `~/admin/TODO.md` |

When a QA check surfaces something that turns out to be pre-existing, still record in the QA section that it was found and that it was diagnosed as not upgrade damage — then carry the actual task to `~/admin/TODO.md` rather than tracking it in both places. Note also that this repo is **public**: see `.gitignore` for the `*.local.md` / `*.local.txt` convention that keeps host-detail and reconnaissance material unpublished.

## System context (relevant when drafting commands or configs)

- **OS:** Fedora 42 → 43, SELinux enforcing (targeted policy)
- **Desktop:** KDE Plasma 6
- **GPU:** AMD Radeon PRO WX 7100 — amdgpu open-source driver only (no AMD proprietary stack)
- **Package managers:** dnf (upgrading to DNF 5 on F43), snap
- **Containers:** 17 Docker Compose services under `/opt/containers/`, each managed by a systemd service + optional backup timer
- **Kasm Workspaces 1.18.1** at `/opt/kasm` — not managed by docker-compose or dnf
- **Backup:** `~/bin/backup.sh` via systemd rsync mirrors; excludes Docker `overlay2`/volumes

## Adding new content

- New upgrade or migration: create a new subdirectory named `fedora-<from>-to-<to>-upgrade/` or `topic-<name>/`.
- New container runbook: add `RUNBOOK.md` inside `/opt/containers/<name>/` on the host (not in this repo), then update `notes-container-inventory.md` to reflect its status.
- Resolved issues: update the relevant checklist file in-place; do not create separate "fixed" files.
