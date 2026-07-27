# TODO

Human-owned task list for this host. Delete this file once every item is checked off.

## Decommission Glean

**Raised:** 2026-07-27 — Glean was reported as decommissioned, but it is still fully
running on this host. Nothing has actually been torn down.

**Current state (verified 2026-07-27):**

| Item | State |
|------|-------|
| Containers | `glean-web`, `glean-worker`, `glean-admin`, `glean-backend`, `glean-postgres`, `glean-redis` — all **Up** and healthy |
| `glean.service` | **enabled** (starts the stack at boot via `docker compose up -d`) |
| `glean-purge.timer` | **enabled**, next trigger 23:00 nightly |
| `glean-purge.service` | **failed** (`203/EXEC`) — SELinux denies `init_t` execute on `glean-purge-articles.sh` (labeled `container_file_t`) |
| Ports | `28805` (glean-web), `28806` (glean-admin) — both bound to `127.0.0.1` |
| DNS / vhosts | `glean.kevininscoe.com`, `glean-admin.kevininscoe.com` — live in `/etc/httpd/conf.d/ssl.conf` (lines ~471–510) |
| Docker volumes | `glean_postgres_data`, `glean_redis_data`, `glean_glean_logs`, `glean_postgres_test_data`, `glean_redis_test_data`, plus legacy `glean_milvus_data`, `glean_milvus_etcd_data`, `glean_milvus_minio_data` |
| Service catalog | Both FQDNs present, status `TBD` |
| On-disk | `/opt/containers/glean/` (compose, `.env`, `RUNBOOK.md`, `backup-glean.sh`, `scripts/`) and source repo `~/Projects/glean` |

The `glean-purge.service` failure resolves itself once the units are removed in step 2 —
no SELinux relabel is needed if Glean is going away.

### Steps

- [ ] Confirm Glean is genuinely being retired, and export/preserve any wanted data (OPML feed list, starred/bookmarked articles) before anything is deleted
- [ ] Remove the purge job: `sudo systemctl disable --now glean-purge.timer`, then delete `/etc/systemd/system/glean-purge.{service,timer}` and `sudo systemctl daemon-reload`
- [ ] Stop and disable the stack: `sudo systemctl disable --now glean.service`, then delete `/etc/systemd/system/glean.service` and reload
- [ ] Take a final backup snapshot of the Docker volumes (`backup-glean.sh`) and verify it is readable before destroying anything
- [ ] Tear down containers and volumes: `cd /opt/containers/glean && sudo docker compose down -v` (this also clears the legacy `milvus` volumes if still attached; remove any orphans with `docker volume rm`)
- [ ] Remove the two Apache vhosts from `/etc/httpd/conf.d/ssl.conf`, then `sudo apachectl configtest` and reload
- [ ] Retire the TLS cert coverage for `glean.kevininscoe.com` and `glean-admin.kevininscoe.com` (see `~/admin/certbot-request.sh`)
- [ ] Delete the `glean` and `glean-admin` DNS records at Linode
- [ ] Release ports `28805` and `28806` per the `portman.md` directive
- [ ] Update the service catalog (`~/Projects/private/fedora-dashboard/kevins-federated-unix-universe-services.md`) per the `service-catalog.md` directive — remove or mark both FQDNs decommissioned
- [ ] Remove the Homepage dashboard tile for Glean, if present
- [ ] Archive or delete `/opt/containers/glean/` and decide the fate of `~/Projects/glean`
- [ ] Log the decommission in `~/ai/fedora/CHANGELOG.md` and commit in `~/ai`
- [ ] Update `fedora-42-to-43-upgrade/notes-container-inventory.md` — the container count drops from 17
