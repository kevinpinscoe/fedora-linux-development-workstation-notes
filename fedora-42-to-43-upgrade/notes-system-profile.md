---
name: System Profile
description: Hardware, OS, disk layout, and key services on the main home server
type: user
originSessionId: ede1e2fc-66bc-4f7d-96c3-643bbfd91d37
---
## Host: FLDW

**Hardware:**
- GIGABYTE B550 AORUS ELITE AX V2 motherboard
- AMD Ryzen 5 5500 6-core
- 128 GB RAM (4x32 DDR4 G.SKILL Trident Z RGB)
- AMD Radeon PRO WX 7100 GPU
- Kingston NV3 2 TB NVMe SSD → root OS (nvme0n1)
- WD8002FZBX 8 TB Black HDD → /home (sda1)

**OS:** Fedora 42 (Adams) — **EOL as of 2026-07-31, updates repo archived**; upgrading to Fedora 43
on 2026-08-01. Kernel `6.19.14-108.fc42.x86_64`, x86_64. F43 will land on kernel 7.1.x.
**Desktop:** KDE Plasma 6 (plasmashell 6.6.4)

**Disk layout:**
- nvme0n1p1 → / (root)
- nvme0n1p3 → /boot/efi
- nvme0n1p4 → /boot
- sda1 → /home (7.3T)
- nvme1n1p1 → /root_backup (mirror)
- nvme1n1p4 → /boot_backup (mirror)
- sdb1 → /home_backup (mirror)

**Backups:** ~/bin/backup.sh via systemd (rsync mirrors root→/root_backup, /boot→/boot_backup, /home→/home_backup). Excludes docker overlay2, volumes, containers dirs.

**SELinux:** enforcing (targeted policy), `selinux-policy-42.24-1.fc42`
**Apache:** httpd 2.4.66 with SSL vhosts in /etc/httpd/conf.d/ssl.conf; the FLDW's main DNS record → Tailscale IP
**Containers:** Fedora `moby-engine` 29.4.2 (**not** Docker CE), `docker-cli` 29.4.2,
`docker-compose` 5.1.2, `containerd.io` 2.2.4 — 44 Compose stacks / 73 running containers
**Snapd:** installed and active — snaps: bare, core, core20, core22, core24, ffmpeg-2404,
gnome-42-2204, gnome-46-2404, gtk-common-themes, mesa-2404, obs-studio, shortwave, snapd
**Config management:** `salt`, `salt-master`, `salt-minion` 3007.5 — **expected to break on F43**
(Python 3.14 incompatibility; breakage accepted 2026-07-31)

**DNS:** <dns-name-redacted> CNAMEs → FLDW (Tailscale tailnet IP)
