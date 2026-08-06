---
title: RUNBOOK.md — Fedora KDE Plasma Desktop (kevin workstation)
tags: [runbook, operations]
vault_link: runbooks/home-kinscoe-projects-public-fedora-notes-kde.md
source_path: /home/kinscoe/Projects/public/fedora-notes/kde/RUNBOOK.md
---

> 📓 Indexed in the PKM knowledge vault at `runbooks/home-kinscoe-projects-public-fedora-notes-kde.md` (symlink → this file).
# RUNBOOK.md — Fedora KDE Plasma Desktop (kevin workstation)

## Metadata

| Field | Value |
|---|---|
| **Owner** | Kevin Inscoe |
| **Last Updated** | 2026-06-28 |
| **Last Tested** | 2026-06-28 |
| **Expected Duration** | N/A — reference + ad-hoc procedures |
| **Risk Level** | Low (desktop-session operations; no data risk) |
| **Repo** | git@github.com:kevinpinscoe/fedora-notes.git |

---

## Purpose

> Operational reference for the **KDE Plasma desktop environment** on this Fedora 42
> workstation (`kevin.kevininscoe.com`, AMD Ryzen 5 5500, 128 GB RAM). Covers the desktop
> session itself — Plasma/KWin, global shortcuts, KRunner, audio/network plumbing as the
> desktop sees it — and recovery for a wedged session.
>
> This is **not** an app runbook (those live in `~/Projects/private/app-configuration/apps/*`)
> and **not** the host automation runbook (systemd backup/sync timers live in
> `~/ai/fedora/RUNBOOK.md`).

---

## When to Use This Runbook

- **Use when:** the Plasma desktop misbehaves (frozen panel, KWin compositor glitch, missing
  tray), you need to add/inspect a **global keyboard shortcut**, or you're re-deriving how the
  desktop is wired (session type, package managers, shortcut config locations).
- **Do NOT use when:** the problem is a specific application (see that app's runbook), a
  self-hosted container (`/opt/containers/`), or a background automation job (`~/ai/fedora/`).

---

## Prerequisites

- [ ] Logged into the KDE Plasma session as `kinscoe` (verify: `echo $XDG_CURRENT_DESKTOP` → `KDE`)
- [ ] A terminal independent of Plasma (a TTY via `Ctrl+Alt+F3`, or Konsole already open) for
      session-recovery steps — restarting Plasma can momentarily kill panel-launched terminals
- [ ] Required tools present (all stock): `plasmashell`, `kwin_wayland`, `kquitapp6`,
      `systemsettings`, `krunner`, `kded6`

---

## Stack

| Component | Details |
|---|---|
| **OS / Release** | Fedora release 42 (Adams), kernel 6.x |
| **Desktop** | KDE Plasma **6.6.4**, KWin **6.6.4** |
| **Session type** | **Wayland** (`XDG_SESSION_TYPE=wayland`) — note: no X11 tools like `xdotool` |
| **Audio** | PipeWire (`/usr/bin/pipewire`) + WirePlumber; control via `pactl` |
| **Network** | NetworkManager (`nmcli`); Tailscale (`/usr/bin/tailscale`) for tailnet |
| **Package managers** | `dnf` (system), `flatpak`, `snap`, `mise` (per-tool runtimes). Homebrew is configured per `~/CLAUDE.md` but not on this shell's PATH |
| **Global shortcuts config** | `~/.config/kglobalshortcutsrc` (mode 0600) |
| **Custom launchers** | `~/.local/share/applications/*.desktop` + scripts in `~/.local/bin/` |

---

## Key Procedures

### Procedure A — Quick-capture Obsidian note (`Meta+N`)

**What it does:** From anywhere in the desktop, `Meta+N` (Super/Windows key + N) opens the
**Notes** Obsidian vault (`~/notes/Notes`) and creates a fresh, timestamp-named note. Because
it uses `Meta+N` rather than `Ctrl+N`, it does **not** collide with the `Ctrl+N`
("new window/tab/file") shortcut used by Obsidian itself or any other app.

**Moving parts:**

| Piece | Path / Value |
|---|---|
| Launcher script | `~/.local/bin/obsidian-new-note` |
| Desktop entry | `~/.local/share/applications/obsidian-new-note.desktop` |
| Key binding | `~/.config/kglobalshortcutsrc` → `[services][net.local.obsidian-new-note.desktop]` → `_launch=Meta+N` |
| Vault | `Notes` → `/home/kinscoe/notes/Notes` (registered in `~/.config/obsidian/obsidian.json`) |
| URI handler | `x-scheme-handler/obsidian` → `obsidian-appimage.desktop` |

The script is a one-liner that invokes Obsidian's URI scheme:

```bash
#!/usr/bin/env bash
# Open the "Notes" Obsidian vault and start a fresh, uniquely-named note.
exec xdg-open "obsidian://new?vault=Notes&name=$(date +%Y-%m-%d_%H%M%S)"
```

**Test without the hotkey:**

```bash
~/.local/bin/obsidian-new-note
```

**Expected output:** Obsidian raises on the **Notes** vault with a new empty note named like
`2026-06-28_143905`.

**If this fails:**
- *Nothing opens* → confirm the URI handler: `xdg-mime query default x-scheme-handler/obsidian`
  should return `obsidian-appimage.desktop`. Re-register with
  `xdg-mime default obsidian-appimage.desktop x-scheme-handler/obsidian` if blank.
- *Wrong vault / "vault not found"* → confirm `Notes` is still registered:
  `grep -o '/home/kinscoe/notes/Notes' ~/.config/obsidian/obsidian.json`. If the vault was
  renamed/moved, update `vault=Notes` in the script.

**To (re)create the binding from scratch** (e.g. on a fresh machine):
1. Recreate the script + desktop entry (contents above; `chmod +x` the script).
2. System Settings → **Keyboard → Shortcuts → Add New → Command or Script…** → command
   `/home/kinscoe/.local/bin/obsidian-new-note` → set shortcut to **`Meta+N`** → **Apply**.
3. Verify the entry landed: `grep -A1 'obsidian-new-note' ~/.config/kglobalshortcutsrc`.

---

### Procedure B — Restart the Plasma shell (frozen panel / tray)

**Why:** Recovers a hung panel, system tray, or desktop without logging out.

```bash
kquitapp6 plasmashell && kstart plasmashell
# fallback if kstart is unavailable:
# kquitapp6 plasmashell; setsid plasmashell >/dev/null 2>&1 &
```

**Success criteria:** Panel, tray, and desktop redraw within a few seconds.

---

### Procedure C — Restart the KWin compositor (visual glitches, no window decorations)

**Why:** Fixes compositor artifacts, stuck windows, or broken effects on Wayland. On Wayland
KWin **is** the session — replace it in place rather than killing it:

```bash
kwin_wayland --replace &
```

**If this fails:** as a last resort, log out and back in (Application Launcher → Leave →
Log Out). Avoid `kill -9` on `kwin_wayland` — it can take the whole Wayland session down.

---

### Procedure D — Inspect / manage global shortcuts

```bash
# See all bound global shortcuts
less ~/.config/kglobalshortcutsrc

# GUI editor
systemsettings kcm_keys      # or: System Settings → Keyboard → Shortcuts
```

**Note:** `~/.config/kglobalshortcutsrc` is read by `kglobalacceld` (the KGlobalAccel daemon,
started by `kded6`). Hand-editing works for existing components but **new** command/launcher
shortcuts should be added via System Settings so they register with the daemon correctly. After
a manual edit, reload with: `kquitapp6 kglobalaccel && kded6 &` (or just re-login).

---

### Procedure E — KRunner won't open (`Alt+F2` / `Alt+Space`)

```bash
kquitapp6 krunner 2>/dev/null; krunner &
```

**Success criteria:** The search overlay appears on the next `Alt+Space`.

---

## Verification

```bash
# Desktop session sanity check
echo "Desktop=$XDG_CURRENT_DESKTOP Session=$XDG_SESSION_TYPE"
plasmashell --version && kwin_wayland --version
grep -A1 'obsidian-new-note' ~/.config/kglobalshortcutsrc
```

**Expected output:**
```
Desktop=KDE Session=wayland
plasmashell 6.6.4
kwin 6.6.4
[services][net.local.obsidian-new-note.desktop]
_launch=Meta+N
```

**Success criteria:** Plasma + KWin report 6.6.x, session is `wayland`, and the `Meta+N`
Obsidian binding is present.

---

## Rollback Procedure

To remove the `Meta+N` Obsidian shortcut and its launcher:

1. System Settings → Keyboard → Shortcuts → find **New Obsidian Note (Notes vault)** → remove
   the binding (or delete the `[services][net.local.obsidian-new-note.desktop]` block in
   `~/.config/kglobalshortcutsrc`).
2. `rm ~/.local/bin/obsidian-new-note ~/.local/share/applications/obsidian-new-note.desktop`
3. `update-desktop-database ~/.local/share/applications`
4. Re-login (or `kquitapp6 kglobalaccel && kded6 &`) so the daemon drops the binding.

---

## Escalation

| Condition | Contact | How |
|---|---|---|
| Session unrecoverable after Plasma/KWin restart | Self (host owner) | Reboot; if persistent, boot to TTY (`Ctrl+Alt+F3`) and inspect `journalctl --user -b` |
| Suspected GPU/Wayland driver regression after `dnf upgrade` | Self | Cross-check `~/Projects/public/fedora-notes/fedora-42-to-43-upgrade/` notes before further upgrades |

---

## Related Runbooks

- [`../RUNBOOK.md`](../RUNBOOK.md) — fedora-notes root runbook (index)
- [`../garage/RUNBOOK.md`](../garage/RUNBOOK.md) — Garage S3 backend
- `~/ai/fedora/RUNBOOK.md` — Fedora workstation **background automation** (systemd timers)
- `~/Projects/private/app-configuration/apps/obsidian/RUNBOOK.md` — Obsidian app config
- `~/SELINUX_SETUP.md` / `../SELINUX_SETUP.md` — SELinux enforcing-mode rules for this host

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `Meta+N` does nothing | Binding lost after re-login or config reset | Re-run Procedure A step 2; verify with the grep in Verification |
| `Meta+N` opens wrong vault | Vault renamed/moved in Obsidian | Update `vault=Notes` in `~/.local/bin/obsidian-new-note` |
| Panel/tray frozen | `plasmashell` hung | Procedure B |
| Window decorations missing / tearing | KWin compositor glitch | Procedure C |
| `Alt+Space` / `Alt+F2` dead | KRunner crashed | Procedure E |
| A global shortcut "sticks" or double-fires | Stale `kglobalacceld` state | `kquitapp6 kglobalaccel && kded6 &`, or re-login |
| `xdotool`/`xdg-open`-style X11 tricks fail | This is a **Wayland** session | Use KWin/Plasma-native tooling; X11 automation tools won't grab/inject keys |

---

## Logs

```bash
# Desktop session logs (current boot, user scope)
journalctl --user -b

# Narrow to Plasma / KWin / shortcut daemon
journalctl --user -b -t plasmashell -t kwin_wayland -t kglobalaccel
```

---

## Maintenance Notes

- **Last game-day test:** 2026-06-28 — `Meta+N` Obsidian quick-capture verified working.
- **Next scheduled review:** after the Fedora 42 → 43 upgrade (see
  `../fedora-42-to-43-upgrade/`) — Plasma version and Wayland driver stack will change.
- **Known drift risks:**
  - Fedora major upgrade bumps Plasma/KWin versions and may reset KDE configs.
  - KDE config resets can drop custom entries in `~/.config/kglobalshortcutsrc`.
  - Obsidian vault path changes break the `Meta+N` launcher (hard-coded `vault=Notes`).
