# Operations: upgrades, backups, and recovery

These commands assume you have completed [the README](../README.md) and are in `~/deployments/docklantern`. Keep a current copy of the password and a verified off-host database backup.

**Using registered Compose projects?** If you enabled the optional, local `compose.projects.yaml` overlay described in [monitoring-first-run.md](monitoring-first-run.md), append `-f compose.projects.yaml` to **every** `docker compose` command on this page, after the two existing `-f` flags. This especially matters for `up -d` and `run`: otherwise you could recreate DockLantern without its read-only source mounts and `-roots` allowlist. The overlay is excluded from Git and must be maintained on the host.

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

## Beta.2 → beta.3 upgrade checklist (not yet authorized)

**The repository currently pins beta.2. Do not change the image to `v0.1.0-beta.3` until the beta.3 release workflow has actually published a tested image and the release is approved.**

Beta.3 introduces automatic monitoring enrollment from target Compose service labels and a **SQLite schema version 3**. A one-way database migration means that simply changing the image back to beta.2 is not a safe rollback.

1. Capture a **consistent beta.2 SQLite backup** while the existing beta.2 app is running (procedure below), verify `PRAGMA integrity_check` returns `ok` on the backup, and copy the file to protected storage outside the VPS.
2. Record the current beta.2 image tag/digest, hostname, and Compose flags. **If you currently use `compose.projects.yaml`, keep passing that third overlay when upgrading.** Do not accidentally remove legacy source mounts or registrations.
3. Verify the beta.3 image digest, multi-platform published manifests, reviewed GitHub CI/security jobs and release notes.
4. Only after operator approval, update the exact app image tag/digest in `compose.yaml`. Validate with `docker compose -f compose.yaml -f compose.discovery.yaml config -q`, adding `-f compose.projects.yaml` if already used.
5. Pull/recreate **only DockLantern**, without restarting Traefik or your other Compose applications. Check readiness, original beta.2 registration/history, and new Services/Settings/Event History.
6. On a separately managed test Compose service (e.g., hello-js), add `com.docklantern.monitor: "true"` to that service's Compose labels and recreate **only that target service**. The next scan should enroll it automatically without editing the DockLantern Compose project.
7. Test global default, label interval, GUI override/reset and pause/resume; verify no DockLantern-initiated container pulls/restarts and that Docker's socket proxy still denies mutation methods.

**Rollback:** stop DockLantern, restore your **entire pre-upgrade beta.2 SQLite snapshot** into the stopped Docker volume (move aside `.db`, `-wal`, `-shm` files before copying), select the exact prior beta.2 image digest/tag and start a single app process. Confirm all registrations, policies, history and readiness. **Do not run an older binary against an already migrated beta.3 database.**

Use the [beta.3 first-run guide](monitoring-first-run.md) after a beta.3 image is installed. Existing source-mounted policies remain supported via the [legacy beta.2 guide](legacy-manual-registration.md). The [application release checklist](https://github.com/docklantern/docklantern/blob/main/docs/deployment/beta3-release-checklist.md) records the CI and manual readiness gates.

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
