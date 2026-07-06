---
title: Garage RUNBOOK
tags: [runbook, operations]
vault_link: runbooks/home-kinscoe-fedora-notes-garage.md
source_path: /home/kinscoe/fedora-notes/garage/RUNBOOK.md
---

> 📓 Indexed in the PKM knowledge vault at `runbooks/home-kinscoe-fedora-notes-garage.md` (symlink → this file).
# Garage RUNBOOK

Operational reference for the self-hosted Garage S3-compatible object store running in the `garage` Docker container on this host.

Endpoint: `https://garage.kevininscoe.com`

---

## Creating a bucket

```bash
docker exec garage /garage bucket create <bucket-name>
```

Example:

```bash
docker exec garage /garage bucket create mail1.kevininscoe.com
```

`bucket create` only creates the bucket — **it does not create or print any access/secret keys**. Keys are managed independently and granted to buckets in a separate step (see below).

Inspect a bucket:

```bash
docker exec garage /garage bucket info <bucket-name>
docker exec garage /garage bucket list
```

---

## Creating an access key (and capturing the secret)

Keys are decoupled from buckets in Garage. Create a key, then grant it permissions on the bucket(s) it should access.

```bash
docker exec garage /garage key create <key-name>
```

**The secret key is printed exactly once at creation time. Capture it immediately — Garage does not store it in retrievable form afterward.**

Output looks like:

```
Key name: <key-name>
Key ID:   GK<...>
Secret key: <...>      ← save this NOW
```

If you miss the secret, you cannot recover it. Your only options are:
- `docker exec garage /garage key delete <key-name>` and recreate it (existing bucket grants are lost — re-grant after).
- `docker exec garage /garage key import` a new key with a known secret (only useful for migrating an existing access/secret pair into Garage).

Inspect keys:

```bash
docker exec garage /garage key list
docker exec garage /garage key info <key-name>      # shows access key ID, never the secret
```

---

## Granting a key access to a bucket

```bash
docker exec garage /garage bucket allow \
  --read --write --owner \
  <bucket-name> \
  --key <key-name>
```

Permission flags (combine as needed):
- `--read` — list and get objects
- `--write` — put and delete objects
- `--owner` — manage bucket aliases, website settings, and CORS

For an OpenTofu state backend, `--read --write --owner` on the project bucket is the typical grant.

Revoke:

```bash
docker exec garage /garage bucket deny \
  --read --write --owner \
  <bucket-name> \
  --key <key-name>
```

---

## End-to-end: new bucket + new key (the common case)

```bash
# 1. Create the bucket
docker exec garage /garage bucket create my-project

# 2. Create the key — SAVE THE SECRET FROM THIS OUTPUT
docker exec garage /garage key create my-project-key

# 3. Grant the key full access to the bucket
docker exec garage /garage bucket allow \
  --read --write --owner \
  my-project \
  --key my-project-key

# 4. Verify
docker exec garage /garage bucket info my-project
```

Then add the access key ID and secret to `~/.aws/config` as a named profile per the OpenTofu IaC standards directive (`~/ai/directives/opentofu-iac-standards.md`).
