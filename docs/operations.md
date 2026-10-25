# Operations: upgrades, backups, and recovery

These commands assume you have completed [the README](../README.md) and are in `~/deployments/docklantern`. Keep a current copy of the password and a verified off-host database backup.

## Common commands

Show the services managed by this Compose project:

```bash
docker compose -f compose.yaml -f compose.discovery.yaml ps
```

Show logs (the `--tail` option keeps output short):

```bash
docker compose -f compose.yaml -f compose.discovery.yaml logs --tail=100 docklantern docker-socket-proxy
```

Start the stack:

```bash
docker compose -f compose.yaml -f compose.discovery.yaml up -d --no-build
```

Stop the DockLantern app and its socket proxy without deleting the SQLite volume:

```bash
docker compose -f compose.yaml -f compose.discovery.yaml stop
```

Restart **just the DockLantern app** after changing the password or local settings:

```bash
docker compose -f compose.yaml -f compose.discovery.yaml up -d --force-recreate --no-deps docklantern
```

All these commands leave the separately deployed Traefik, hello services, and unrelated workloads alone.

## Upgrade DockLantern safely

1. Create and verify a database backup first (see below).
2. Review the new image release and update `services.docklantern.image` in `compose.yaml` to a specific tested version, not `latest`.
3. Validate, pull, and recreate just the app:

```bash
docker compose -f compose.yaml -f compose.discovery.yaml config -q
docker compose -f compose.yaml -f compose.discovery.yaml pull docklantern
docker compose -f compose.yaml -f compose.discovery.yaml up -d --no-deps docklantern
docker compose -f compose.yaml -f compose.discovery.yaml ps
docker compose -f compose.yaml -f compose.discovery.yaml logs --tail=100 docklantern
```

Test the login page, `/readyz`, inventory and registered projects. **Do not roll back application versions against an incompatible migrated database**; use your tested restore plan if necessary.

## Consistent SQLite backup while running

DockLantern stores its database in the named Docker volume `docklantern_docklantern_data`. The application supports `-backup`, which makes a consistent SQLite snapshot of committed transactions (including WAL changes). Do **not** simply `cp` the live database file.

The following example creates a root-owned directory for backup output, then runs a **one-time backup command** in a new short-lived container. It does not start another monitoring scheduler.

```bash
# Prepare a private backup directory, owned by root.
sudo install -d -m 0700 -o root -g root backups

# Produce a timestamped backup using the application's SQLite backup mode.
docker compose -f compose.yaml -f compose.discovery.yaml run --rm --no-deps \
  -v "$(pwd)/backups:/data/backups" \
  docklantern -db /data/docklantern.db \
  -backup "/data/backups/docklantern-$(date -u +%Y%m%d-%H%M%S).sqlite"

# Inspect backup files on the host.
sudo ls -lh backups/
```

The output file must not already exist. Copy the backup to **another host or secure storage location**; a backup on the same VPS is not disaster recovery. Limit access because SQLite may contain project metadata, observations and operational history.

Test integrity on a separate machine with SQLite:

```bash
sqlite3 /path/to/downloaded-backup.sqlite 'PRAGMA integrity_check;'
```

Expected result: `ok`. Use a real local file path on the checking machine, not the literal example path.

## Restore from a known-good SQLite backup

**Restoring overwrites application state.** Plan downtime and confirm the file is the right backup. Do not copy a database over live SQLite processes.

1. Stop the DockLantern app with `docker compose -f compose.yaml -f compose.discovery.yaml stop docklantern`. Ensure no other process uses this SQLite volume.
2. Save an independent rollback copy of the current database *and* any `-wal` and `-shm` files.
3. Verify the backup with `PRAGMA integrity_check;` (expect `ok`) before copying it.
4. In the persistent volume, move aside the original `docklantern.db` **and both accompanying journal files** (`docklantern.db-wal`, `docklantern.db-shm`). Do not mix the restored file with stale journal files.
5. Copy the verified standalone backup into the volume at `/data/docklantern.db`, with owner and mode that match the existing container's volume access.
6. Start a **single** DockLantern app instance and verify `/readyz`, registrations, monitoring policies, and history.

If you are unfamiliar with manipulating Docker named-volume contents, do the recovery on a **test machine** first. Volume mount points vary across rootful/rootless Docker installations. See [the application backup documentation](https://github.com/docklantern/docklantern/blob/main/docs/deployment/beta-readiness.md).

## Password rotation

1. Generate a new secret file using the same secure steps in the README.
2. Ensure it is `root:root` and mode `0600` for rootful Docker.
3. Recreate **only** the DockLantern service using `up -d --force-recreate --no-deps docklantern`.
4. Confirm the new credential works and the old credential does not.

## What a full server rebuild needs

This repository does **not** contain the full host configuration. Record separately the installed Docker and Compose versions, external Docker network definitions, Traefik configuration and TLS certificates, DNS/CDN records, firewall rules, and IPv6 routing. Never commit private keys, admin passwords, registry credentials, or database backups.

Safe inspection commands (review results for sensitive host metadata before sharing):

```bash
docker version
docker compose version
docker network inspect web
docker network inspect docklantern-ipv6-egress
docker inspect traefik --format '{{json .Config.Cmd}}'
```

## Important data-loss warning

Avoid `docker compose down -v`. The `-v` flag removes volumes, including the persistent database. A plain `docker compose stop` is the safer routine stop operation.
