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
implicitly created by Docker.

The overlay only adds a bind mount with:
- source: `/var/lib/docklantern/handoff/receipts`
- target: `/run/docklantern/executor-receipts`
- `read_only: true`, `bind.create_host_path: false`

It never mounts the executor journal or protected registrations, raw Compose
sources, executor binary, root Docker socket or any host mutable authority.
It does not change the existing proxy's `POST=0` restriction.

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
pass and the operator authorizes the container recreation, use the
`compose.receipts.yaml` overlay **last** in the invocation's `-f`
sequence. For example, the operator-approved deployment command would use:

```bash
# EXAMPLE ONLY — do not run without approval and a reviewed image pin.
docker compose -f compose.yaml -f compose.discovery.yaml \
  -f compose.receipts.yaml up -d --no-deps docklantern
```

If using the separate legacy `compose.projects.yaml` overlay, retain that
overlay in the sequence before `compose.receipts.yaml`; otherwise you may
remove existing read-only project mounts while recreating the monitor.

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
