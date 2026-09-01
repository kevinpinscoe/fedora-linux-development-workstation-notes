# SELinux & Container Setup — kinscoe@fedora

## System Overview

- **OS**: Fedora 42, KDE Plasma
- **SELinux**: Enforcing mode
- **User**: kinscoe (`/home/kinscoe`)
- **Container runtime**: Docker (docker-compose)

---

## Known Conflicts & Their Resolutions

### 1. Python File Server Container (`/home/containers/home_file_server`)

**Purpose**: LAN file server. Exposes `/home/kinscoe` as a read-only HTTP browse interface on port 2222. Used to access home directory files from other machines on the local network.

**Historical problem**: The original volume mount used the `:Z` flag:
```
volumes:
  - "${HOME}:/srv:ro,Z"
```
The `:Z` flag tells the container runtime to relabel the host source directory with a private MCS SELinux label (`container_file_t:s0:c193,c817`). Since the source was `$HOME`, this relabeled the **entire home directory**, breaking:
- `snapd` (which expects `user_home_dir_t` on home dirs)
- `~/snap/` user data directories
- Any other process that checks SELinux labels on home

**Resolution**: Use `security_opt: label:disable` instead of `:Z`. This disables SELinux label enforcement for this specific container without touching host filesystem labels. Safe for a read-only LAN-only server.

```yaml
services:
  recordings:
    image: python:3.13-alpine
    container_name: home_file_server
    command: ["python", "-m", "http.server", "8000", "--bind", "0.0.0.0"]
    ports:
      - "0.0.0.0:2222:8000"
    volumes:
      - "${HOME}:/srv:ro"
    working_dir: /srv
    restart: unless-stopped
    security_opt:
      - label:disable
```

**Expected label on /home/kinscoe**:
```
unconfined_u:object_r:user_home_dir_t:s0
```
If it ever shows `container_file_t`, run:
```bash
sudo restorecon -Rv /home/kinscoe
```

---

### 2. yt-dlp — Native Install, Not Snap

**Historical problem**: `yt-dlp` was installed as a snap. Snap user data lives in `~/snap/yt-dlp/`, which inherits whatever label is on `$HOME`. When the python_file_server container applied `:Z` to `$HOME`, snap data got relabeled and `snapd` was denied access to it, generating dozens of AVC denials per minute.

**Resolution**: Removed the snap, installed via pip:
```bash
sudo snap remove yt-dlp
pip install --user yt-dlp
# Binary lives at ~/.local/bin/yt-dlp
```

pip-installed binaries in `~/.local/` carry normal `user_home_t` and are unaffected by container label operations.

---

### 3. `/root_backup/dev/snapshot` — Backup Mount Device

**Problem**: `setroubleshootd` (`setroubleshootd_t`) was denied `getattr` on `/root_backup/dev/snapshot`, a character device on `nvme0n1p1` labeled `default_t`. This was a hard block (`permissive=0`).

**Cause**: `/root_backup` is a backup filesystem mount. Its `/dev/snapshot` device node was created with a default label rather than a proper device type.

**Resolution**:
```bash
sudo semanage fcontext -a -t fixed_disk_device_t '/root_backup/dev/snapshot'
sudo restorecon -v /root_backup/dev/snapshot
```

---

### 4. `setroubleshootd` Capability Denials (sys_admin, sys_resource)

**Problem**: `SetroubleshootP` process was denied `CAP_SYS_ADMIN` and `CAP_SYS_RESOURCE` capabilities (`permissive=0`).

**Cause**: SELinux policy gap in newer Fedora releases. `setroubleshootd_t` policy doesn't grant these capabilities needed by the setroubleshoot process pool workers.

**Resolution**: Try updating first (may have a policy fix):
```bash
sudo dnf update selinux-policy\* setroubleshoot\*
```
If no update available or denials persist, install a local module (applied 2026-05-02):
```bash
sudo ausearch -m avc -c SetroubleshootP | audit2allow -M setroubleshoot-fix
sudo semodule -i setroubleshoot-fix.pp
```
Active module name is `setroubleshoot-fix` (installed 2026-05-02; `selinux-policy-42.24-1.fc42` had no fix).

---

### 5. `rsync_t` `dac_override` Denials (backup services)

**Problem**: `rsync` denied `{ dac_override }` when run from `backup-<remote-host-c>.service` and `backup-<remote-host-b>.service`. Root-owned files (e.g. `/var/log`, `/root`) silently skipped with no non-zero exit code.

**Cause**: The base `rsync_t` policy does not grant `dac_override`. Rsync needs it to read files owned by root or other users and to preserve ownership metadata on the receiving end.

**Resolution** (applied 2026-05-02, module name `rsync-backup`):
```bash
sudo ausearch -m avc -c rsync | audit2allow -M rsync-backup
sudo semodule -i rsync-backup.pp
```
Policy rule: `allow rsync_t self:capability dac_override;`

---

### 6. `~/git_repos` labeled `httpd_sys_content_t`

**Problem**: `snapd` was denied read access to `~/git_repos` because it had a web server label.

**Cause**: Likely set manually at some point when testing httpd content serving from home directory.

**Resolution**:
```bash
sudo restorecon -Rv /home/kinscoe/git_repos
```

---

## Container Strategy — Label-Safe Patterns

### Rule: Never use `:Z` or `:z` on `$HOME` directly

| Volume flag | What it does | Safe to use on $HOME? |
|---|---|---|
| `:ro` | Read-only, no label change | ✅ Yes |
| `:z` | Relabels to shared `container_file_t` | ❌ No |
| `:Z` | Relabels to private `container_file_t` | ❌ No |
| (none) | No label change | ✅ Yes |
| `label:disable` via security_opt | Disables SELinux for container | ✅ Yes (trusted containers only) |

### Rule: Containers that need SELinux confinement get a dedicated share directory

```bash
# Create once
mkdir -p ~/container_shares/<app_name>
sudo semanage fcontext -a -t container_file_t '/home/kinscoe/container_shares(/.*)?'
sudo restorecon -Rv ~/container_shares/
```

Then mount `~/container_shares/<app_name>:/data:Z` in the compose file. The `:Z` label operation only touches the subdirectory, not `$HOME`.

---

## Diagnostic Commands

```bash
# Check current AVC denials (today)
sudo ausearch -m avc -ts today | grep -v '^----'

# Check labels on home dir
ls -Z /home/ | grep kinscoe

# Check labels recursively on a specific path
ls -laZ /home/kinscoe/snap/

# Restore default labels on home dir
sudo restorecon -Rv /home/kinscoe

# Watch live AVC denials
sudo tail -f /var/log/audit/audit.log | grep AVC
```

---

## Services Running Containers

| Container | Compose file location | Port | Notes |
|---|---|---|---|
| home_file_server | `/home/containers/home_file_server/` | 2222 | LAN file browse, uses label:disable |

---

*Last updated: 2026-05-02*
