# Beta.5 test-VPS upgrade plan — operator approval required

**Status:** Prepared, **not executed**. This document is a review checklist, not authorization to access or modify the running VPS. The [application beta.5 prerelease](https://github.com/docklantern/docklantern/releases/tag/v0.1.0-beta.5) is published. The deployment repository proposes replacing the beta.4.1 image pin **only for DockLantern**, preserving all other Compose service settings and the restricted socket proxy.

**Release image:** `davidslusser/docklantern:v0.1.0-beta.5@sha256:d36784df4eebc29fb45c3d03441101b0d81078fa971eecfe7e661efc18608c16`

This immutable **multi-platform index digest** was recorded for Docker Hub **and** GHCR by the [successful release workflow](https://github.com/docklantern/docklantern/actions/runs/41239040126), built for `linux/amd64` and `linux/arm64`. Verify the tag/index with a separate registry-capable client before an approved deployment; the workflow logs are evidence of publication, **not proof of registry reachability from an IPv6-only VPS**.

For application acceptance, see [beta.5 test-VPS checklist](https://github.com/docklantern/docklantern/blob/main/docs/deployment/beta5-test-vps-acceptance.md). Track final disposition in [#68](https://github.com/docklantern/docklantern/issues/68) and [#69](https://github.com/docklantern/docklantern/issues/69). The versioned [release notes](https://github.com/docklantern/docklantern/releases/tag/v0.1.0-beta.5) describe the watched-tag and Check Now changes.

## Why backup ordering matters

The published beta.4.1 app uses **SQLite schema v3**. Beta.5 uses transactional **v4** (durable watched image reference) and **v5** (persistent manual Check Now cooldown). Opening a v3 database with the new binary may migrate it to v5. Older beta.4.1 **will not safely open** that upgraded database.

**Take the backup before checking out or applying this PR's Compose image change on the VPS.** The usual `docker compose run --rm --no-deps ... docklantern -backup ...` starts a *new container from the image currently selected by the local Compose file*. If that file already points to beta.5, it may migrate the live volume even if the command was intended solely to create a backup.

If the VPS deployment working tree already points at beta.5, **stop**: verify the actually running image and whether a v3-compatible backup already exists; do not issue the generic backup command against the changed file. Explicitly use a verified, still-available **beta.4.1** backup method/configuration and reconfirm it cannot start the new binary.

## Gate A — baseline and consistent backup, separately approved

These are **operator-run commands after explicit approval**, from the existing `~/deployments/docklantern` directory **while the local Compose configuration still references beta.4.1**. Do not paste secrets, resolved Compose environment, or unredacted Docker metadata into public issues.

```sh
# Read-only baseline. Verify the active overlay set before proceeding.
docker compose -f compose.yaml -f compose.discovery.yaml config --images
docker compose -f compose.yaml -f compose.discovery.yaml ps
CID="$(docker compose -f compose.yaml -f compose.discovery.yaml ps -q docklantern)"
docker inspect "$CID" --format 'Image={{.Config.Image}} ImageID={{.Image}} Created={{.Created}}'
docker exec "$CID" docklantern-inventory -version
```

Before using backup commands, capture the actual immutable IDs for the three monitored hello-service containers, configured registry access/IPv6 egress, and baseline readiness/idle-resource measurements. Verify whether a local `compose.projects.yaml` overlay is in use and include it consistently throughout the process. Do **not** change hello service Compose files, images or labels for this upgrade.

Only **after verifying that the local Compose file still refers to beta.4.1** and confirming backup authorization:

```sh
# Creates a local private backup directory (state-changing; explicit approval required).
sudo install -d -m 0700 -o root -g root backups

# Uses the *existing beta.4.1 Compose image*, not the proposed beta.5 image.
# The tool performs a SQLite online snapshot; never plain-cp the active DB.
docker compose -f compose.yaml -f compose.discovery.yaml run --rm --no-deps \
  -v "$(pwd)/backups:/data/backups" \
  docklantern -db /data/docklantern.db \
  -backup "/data/backups/docklantern-before-beta5-$(date -u +%Y%m%d-%H%M%S).sqlite"
```

Confirm the generated filename, verify `PRAGMA integrity_check` returns `ok` on that **backup file** and record the pre-upgrade database `user_version` (expected 3, verify rather than assume). Securely copy the snapshot **off the VPS**, retaining permissions and a recorded restore location. Decide explicitly whether to perform a separate restore drill; an integrity check **is not** a full disaster recovery test. Do not share backup files.

## Gate B — review the deployment PR, still no live change

- Confirm the [app release](https://github.com/docklantern/docklantern/releases/tag/v0.1.0-beta.5), matching approved commit, completed CI/security/release workflow and immutable digest.
- Review the PR diff: **only** DockLantern's image reference changes in `compose.yaml`. Socket proxy, command, ports (none published), Traefik, secrets, healthcheck, named volume, discovery, external IPv6 egress and target apps remain untouched.
- Validate with `docker compose -f compose.yaml -f compose.discovery.yaml config -q` in a **separate local checkout** or disposable CI first. Do not run `up`, `pull`, `down`, `restart` or `run` against the live beta.5 configuration during this gate.
- Require **separate operator approval** to merge the configuration PR and then **separate explicit approval** to deploy it to the VPS, after the backup and rollback readiness review.

## Gate C — approved test-VPS rollout only

After backup/rollback and release verification, and **only with explicit deployment authorization**, update the VPS checkout to the reviewed deployment commit. Reconfirm any local overlays. The expected deployment changes **only DockLantern**, using its already-pinned immutable Docker Hub index digest; do not execute `docker compose up -d` against all services.

With the correct active Compose overlays and permissions:

```sh
docker compose -f compose.yaml -f compose.discovery.yaml config -q
docker compose -f compose.yaml -f compose.discovery.yaml pull docklantern
docker compose -f compose.yaml -f compose.discovery.yaml up -d --no-deps docklantern
docker compose -f compose.yaml -f compose.discovery.yaml ps
docker compose -f compose.yaml -f compose.discovery.yaml logs --tail=100 docklantern
```

On this host the Docker engine may access Docker Hub whereas GHCR may be inaccessible over IPv6; the proposed deployment therefore continues using the **published Docker Hub image**. Do not change to GHCR during upgrade without explicitly validating reachability.

Do not add `--force-recreate` unnecessarily, rebuild the socket proxy, replace Traefik, prune volumes or restart the monitored hello apps. **Never use `docker compose down -v`.**

## Gate D — post-deploy acceptance

- Verify running image digest, embedded `docklantern-inventory -version`, `/readyz` and current schema v5, preserving policies/events/settings/history from v3.
- Check authenticated Overview, Images, Image Details, Services, Service Details, Activity and Settings; assess keyboard/mobile/tablet readability and CSRF/Origin protection.
- On a safe already-enrolled test service, choose a **same-repository watched tag**, verify Docker-observed image remains unchanged, request **Check Now** and confirm the subsequent worker observation. A `scheduled` response means *queued*, not a successful registry fetch. Confirm cooldown/backoff and safe reset.
- Compare before/after container IDs and immutable image IDs of **hello-py, hello-js and hello-go**; none should be recreated or restarted by DockLantern's monitoring actions.
- Confirm private GET-only Docker socket proxy, no Docker writes, IPv6-only registry access, idle resource usage and representative page latency.
- Record pass/fail evidence and explicitly accepted deferrals in the application [phase-4 issue #68](https://github.com/docklantern/docklantern/issues/68). Do not claim production certification or mark the release complete solely because Compose reports healthy.

## Rollback — use a database snapshot compatible with the old image

**Do not simply change the image pin back to beta.4.1 while retaining a v5 SQLite database.** If the deployment fails, stop **only** DockLantern, recover the pre-upgrade verified v3 SQLite snapshot, and restore beta.4.1's exact prior image/digest with the corresponding approved deployment configuration.

The [operations recovery guide](operations.md#restore-from-a-known-good-sqlite-backup) details moving aside the entire current database **and WAL/SHM companions** before restoring a verified snapshot, then starting a single app instance and validating readiness and history. This is a **state-replacing recovery** and must be explicitly authorized. Back up current post-upgrade state independently before any destructive restore.

**Current state:** beta.5 images are published, but this deployment PR is only a configuration proposal. The live VPS must remain unchanged until all required approvals have been provided.
