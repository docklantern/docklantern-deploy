# First monitored Compose service — label-based onboarding (beta.3)

**Release status:** **v0.1.0-beta.3 is published**, and this deployment repository now pins its verified immutable image manifest. If you have not yet recreated DockLantern using the updated Compose configuration, check the running image before using these steps. Upgraded legacy beta.2 registrations remain supported via the [legacy source registration guide](legacy-manual-registration.md).

This guide assumes a working DockLantern deployment with **`compose.yaml` + `compose.discovery.yaml`**, a private Docker socket proxy, periodic discovery (`-scan-every 60s`) and a running registry monitor. Follow the [README](../README.md) for host/network/TLS prerequisites. All operations in this guide are on the **existing application Compose project**, not in the DockLantern deployment directory.

## 1. Add one monitoring label to the target service

For example, in `hello-js` or `hello-py`'s own Compose file:

```yaml
services:
  web:
    image: davidslusser/hello-js:latest
    labels:
      com.docklantern.monitor: "true"
      # Optional. Omit this to use the default monitoring interval:
      com.docklantern.interval: "10m"
```

Only the `com.docklantern.monitor` label is required. The optional interval accepts whole-second Go duration syntax from `60s` to `24h`. The default is initially five minutes, configurable from **Settings → Monitoring defaults**.

**Do not add these labels to the DockLantern container.** Add them to each *application service you want monitored*. You do **not** need to register Compose sources, mount the app directory into DockLantern, change `.env`, or extend `-roots`.

## 2. Apply the changed target Compose configuration

From that service's own deployment directory, run:

```bash
docker compose config -q
docker compose up -d --no-deps web
```

Compose may recreate **the target application container** to apply its changed labels; that is expected. It must **not** recreate DockLantern. Using `--no-deps` prevents starting dependent services unnecessarily.

To verify actual container labels:

```bash
docker compose ps -q web | xargs -r docker inspect --format '{{index .Config.Labels "com.docklantern.monitor"}}'
```

Expected: `true`. If you have scaled replicas, every replica of the service must have consistent image/platform and monitoring labels.

## 3. Confirm automatic enrollment

With the existing Docker socket proxy and inventory scanner active, DockLantern discovers the service during its next scan (typically within 60 seconds). Open the authenticated HTTPS dashboard and navigate to **Services**. Verify that the Compose project/service appears, with enrollment source **Docker label** and the expected effective interval.

From DockLantern **Settings → Service monitoring**, you can change the interval without restarting DockLantern or the target service:

- **Save override:** takes precedence over a Docker interval label.
- **Reset override:** returns to the Docker interval label, or the global default if the label is absent.
- **Pause / Resume:** changes registry monitoring scheduling only.
- For an **unlabeled** discovered service, **Enable in GUI** explicitly enrolls it (provided it is running with valid image/platform metadata).

Configuration precedence: **GUI override > Docker label > global default**. An interval override alone does not opt a service into monitoring.

## 4. Understand registry and image status

DockLantern reads OCI manifests and compares their digests. `first_seen`, `changed`, and `unchanged` are **remote tag observations**; `changed` does not guarantee a newer software version. Container image assessments require separate verifiable digest provenance and may remain **unknown**.

The Events page shows remote digest changes as **image/platform-scoped events**, not as an arbitrary project/service action. `Webhook pending (0 attempts)` is notification state; it is not a registry-check failure.

DockLantern remains **monitoring-only**: it does not pull, restart, update or redeploy monitored services.

## 5. Disable or remove monitoring

To enforce an explicit opt-out, change the target service label and redeploy **only that service**:

```yaml
labels:
  com.docklantern.monitor: "false"
```

This blocks registry monitoring even if a prior GUI opt-in or legacy policy exists. Removing `true` from a label-owned service also stops label-owned monitoring (unless separately GUI-enrolled). History and settings remain stored for diagnostics.

If an image reference is unsupported, platform metadata is missing, replicas conflict, or label values are invalid, DockLantern shows an inactive/error state and does not activate the watch. A transient discovery-proxy failure must not be mistaken for removing every service.

## Existing beta.2 registrations and future deployment features

An upgraded beta.2 installation retains its **legacy manual registrations, monitoring policies and history**. It is not required to remove any local `compose.projects.yaml` overlay as part of the beta.3 upgrade. Such source mounts remain relevant to older registrations and separately authorized future Compose deployment functionality, **but are not needed for new label-based registry monitoring**.

If you already use manual registration, see [legacy source registration](legacy-manual-registration.md). Before upgrading the running image, take a **consistent SQLite backup** and review the [operations guide](operations.md). A beta.2 binary may not understand the migrated beta.3 SQLite schema; rollback requires the matching pre-upgrade backup, not just changing the image tag.

## Troubleshooting

- **No service in inventory:** Check that `compose.discovery.yaml` is active, the proxy is healthy and `-scan-every` is nonzero; inspect DockLantern's discovery worker status and logs.
- **Service appears, no monitoring policy:** Inspect labels on the **running container**, not just YAML. Check the inactive reason in Settings. `monitor=false`, missing image platform and invalid durations fail closed.
- **No registry result yet:** Check the monitor worker, registry connectivity, credentials for private images, and next-check timestamp. Discovery and registry checking are separate periodic loops.
- **Changed GUI interval not applied:** Verify whether the service is paused or rate-limited; the scheduler preserves registry backoff even after changing the interval.
- **No original Compose files mounted into DockLantern:** That is expected for label-only monitoring.

Never expose the Docker proxy's internal TCP port 2375, and keep access to the dashboard behind trusted HTTPS.
