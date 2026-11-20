# Optional Release-Candidate Deployment Activity receipt overlay

**NOT enabled by default.** This is a Compose-only read-only view of the
host executor's **sanitized receipt JSON**, not a host-executor installation.
Merging this file does **not** authorize source changes, Docker updates, RC
installation, timer activation or receipt mount creation on any VPS.

The main monitor has a historical Activity endpoint that reads the fixed
container path `/run/docklantern/executor-receipts`. Without a trusted
receipt mount the GUI correctly reports unknown/unavailable. For positive
receipt acceptance on an approved test VPS, use
[`compose.receipts.yaml`](../compose.receipts.yaml).

## Host security prerequisites

Only the host administrator, after explicit approval, may provision
`/var/lib/docklantern/handoff/receipts` and its parents to match the
protected host-executor receipt publication policy. The outbox directory must
be **root-owned, mode 0755** and have no group/world-writable ancestors.
Valid published receipt files must be root-owned, single-link, regular files
with exact mode **0644** and the existing canonical receipt schema.
An absent or otherwise unsafe host path must STOP installation, not be
implicitly created by Docker. The outbox is the only public receipt subtree;
never recursively change permissions on the protected handoff or executor tree.

The overlay only adds a bind mount with:
- source: `/var/lib/docklantern/handoff/receipts`
- target: `/run/docklantern/executor-receipts`
- `read_only: true`, `bind.create_host_path: false`

It never mounts the executor journal or protected registrations, raw Compose
sources, executor binary, root Docker socket or any host mutable authority.
It does not change the existing proxy's `POST=0` restriction.

## Provision before enabling any automatic-update timer

The existing VPS's first automatic deployment committed successfully but
published no receipt because the outbox was mode 0700. After an explicitly
approved correction to 0755, the next operation published its receipt
successfully and the GUI displayed it through this overlay. An unchanged tick
does not backfill the missing receipt. Provision and verify the outbox before
the first accepted live update on a clean host.

First provision the protected parent directories according to the reviewed
host executor installation procedure. Only after explicit host provisioning
approval, run the following on the selected host. This creates **only** the
missing outbox leaf, never repairs an existing unsafe directory or changes
its ancestors. The parent tree must already exist and be trusted.

```bash
sudo python3 - <<'PY'
import os
import stat
from pathlib import Path

outbox = Path("/var/lib/docklantern/handoff/receipts")
for parent in reversed(outbox.parents):
    s = parent.lstat()
    if not stat.S_ISDIR(s.st_mode) or s.st_uid != 0 or s.st_mode & 0o022:
        raise SystemExit(f"STOP: unsafe parent {parent}")

try:
    s = outbox.lstat()
except FileNotFoundError:
    os.umask(0o022)
    outbox.mkdir(mode=0o755)
    s = outbox.lstat()

if (not stat.S_ISDIR(s.st_mode) or s.st_uid != 0
        or s.st_gid != 0 or stat.S_IMODE(s.st_mode) != 0o755):
    raise SystemExit("STOP: outbox must be root:root, real directory, 0755")

print("PASS: trusted receipt outbox")
PY
```

If the path already exists with unsafe mode or ownership, inspect it and
obtain separate approval for a narrow correction. Do not automatically
follow a symlink, replace a directory, delete a receipt, or relax parent
permissions. An existing outbox must contain only executor-managed files;
inspect receipt metadata without printing private journal or source data.
On a clean VPS the outbox should initially be empty. Before starting the
timer verify the monitor mount is read-only and the executor's private
journal/registrations are not exposed.

## Validation without deployment

On a checkout with the usual local untracked `.env` and password file,
the following is **configuration-only**:

```bash
docker compose -f compose.yaml -f compose.discovery.yaml \
  -f compose.receipts.yaml config -q

docker compose -f compose.yaml -f compose.discovery.yaml \
  -f compose.receipts.yaml config --format json
```

Review the resulting service volume list: the additional mount must be
read-only and the only target added by this overlay must be the fixed receipts
path. No host directory is created by `config`.

The checked-in `compose.yaml` remains pinned to an older published beta
image until a separate release-image update is explicitly approved. A
configuration validation is **not** an RC deployment, a host-executor update
or proof that historical receipts are visible in the GUI.

## Separately approved RC usage

After the approved host outbox exists, the RC monitor image/backup checks
pass and the operator authorizes the container recreation, preserve the
**entire original ordered Compose file stack** and add the receipt overlay
last. On an existing host verify the exact file order from the running
monitor's `com.docker.compose.project.config_files` and
`com.docker.compose.project.working_dir` labels. Verify project/service,
local image ID and effective Compose service configuration before recreation.
A missing legacy, release or RC image overlay may drop existing mounts or
revert to the old beta image. Compare base and proposed effective
configurations privately (resolved configuration can contain secrets);
only the intended receipt mount may differ.

Example for a fresh approved RC with a reviewed local image overlay
`compose.rc.yaml` (operator-supplied, not a checked-in file):

```bash
# EXAMPLE ONLY — requires separate deployment approval.
# Verify the local RC image and effective image pin first.
docker compose -p docklantern \
  -f compose.yaml -f compose.discovery.yaml -f compose.rc.yaml \
  -f compose.receipts.yaml config -q

docker compose -p docklantern \
  -f compose.yaml -f compose.discovery.yaml -f compose.rc.yaml \
  -f compose.receipts.yaml up -d --no-deps --no-build --pull never \
  --force-recreate docklantern
```

For an existing stack substitute its actual ordered files and project name,
including any legacy `compose.projects.yaml` or release overlay. Do not
recreate the proxy or other services, remove named volumes, pull/build a
different image, or edit the executor-managed target override. Verify
the same image ID, original mounts, environment, networks and named data
volume; compare unrelated container IDs and confirm GUI authentication,
inventory and Activity. Keep using the full file stack for future operations.

The first positive read test should compare: protected executor journal
operation outcome, exact sanitized receipt operation ID and digest, current
Docker container ID/image ID/digest, and the GUI's latest inventory timestamp.
A successful past receipt **must not** be interpreted as current runtime
success. Verify that a missing/stale outbox fails closed, and check the
monitor cannot change any outbox file.

To disable visibility, after separate operator approval, recreate only the
monitor **without** this optional overlay. Do not delete the host receipt
outbox or executor journal as a workaround.

See [application first-release RC acceptance](https://github.com/docklantern/docklantern/blob/main/docs/deployment/first-release-rc-acceptance.md)
for the two-tag test and clean-host signoff.
