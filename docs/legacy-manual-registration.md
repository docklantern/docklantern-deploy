# Legacy beta.2 source-based registration

This document preserves the original manual registration workflow used by **v0.1.0-beta.2**, and remains useful when maintaining an existing source-mounted Compose registration. For **beta.3 and newer** image monitoring, source mounts and `-roots` are **not required**; use [the label-based first-run guide](monitoring-first-run.md) after a beta.3 image has actually been published and selected.

---

# First monitored Compose projects: beginner walkthrough

This optional setup builds on the main [installation guide](../README.md). You must have:
- A working DockLantern dashboard and discovery proxy.
- Two existing Docker Compose projects running on the same host (examples here: `hello-py` and `hello-js`).
- Permission to inspect the projects' original Compose file(s) and any needed `.env`.

**Do not commit actual hostnames, host paths, environment files, or passwords.** Everything below uses generic example paths. Example directory names can be changed to suit your deployment.

## 1. Identify existing Compose sources

Run these commands on the **host**, replacing the example container names if necessary:

```bash
docker inspect hello-py-web-1 --format '{{index .Config.Labels "com.docker.compose.project.working_dir"}}'
docker inspect hello-py-web-1 --format '{{index .Config.Labels "com.docker.compose.project.config_files"}}'
docker inspect hello-js-web-1 --format '{{index .Config.Labels "com.docker.compose.project.working_dir"}}'
docker inspect hello-js-web-1 --format '{{index .Config.Labels "com.docker.compose.project.config_files"}}'
```

Record the **exact absolute paths**. If the labels are absent, identify the source files with the service owner before proceeding. Check each Compose file for `env_file:`, other referenced Compose files, and any secrets. Mount only the files/directories required for monitoring and registration; the read-only mount grants the DockLantern process read access to their contents.

To display only non-commented `.env` **key names** (never post the actual values):

```bash
grep -E '^[A-Za-z_][A-Za-z0-9_]*=' /srv/apps/hello-py/.env | cut -d= -f1
```

Replace `/srv/apps/hello-py` with the actual location on your machine. Variable names alone are not proof there are no secrets; inspect the file privately.

## 2. Create the private registration overlay

Go to the DockLantern deployment directory:

```bash
cd ~/deployments/docklantern
cp compose.projects.example.yaml compose.projects.yaml
nano .env
```

Set the paths for **your host**:

```dotenv
HELLO_PY_PROJECT_DIR=/srv/apps/hello-py
HELLO_JS_PROJECT_DIR=/srv/apps/hello-js
```

Replace the sample paths with the actual directories from Step 1. The sample overlay:
- Binds both directories into DockLantern **read-only**.
- Uses identical source and destination absolute paths because DockLantern validates absolute Compose file locations.
- Prevents Docker from silently creating a missing host directory (`create_host_path: false`).
- Adds both source directories to the explicit `-roots` allowlist.
- Keeps `-scan-every 60s` and `-run-monitor` active.

**Do not modify or mount anything into the Docker socket-proxy service.** The private `compose.projects.yaml` file and `.env` are ignored by Git. The example overlay does not enable Docker update execution.

## 3. Validate and apply (DockLantern only)

```bash
docker compose -f compose.yaml -f compose.discovery.yaml -f compose.projects.yaml config -q
docker compose -f compose.yaml -f compose.discovery.yaml -f compose.projects.yaml up -d --no-deps docklantern
docker compose -f compose.yaml -f compose.discovery.yaml -f compose.projects.yaml ps
```

Expected: Docker Compose validation shows no errors, DockLantern is `Up (healthy)`, and the socket proxy remains up. Existing hello services and Traefik are not restarted.

**Use all three `-f` arguments on future Compose commands that recreate DockLantern.** If you leave off the third overlay, Compose can remove the source mounts and `-roots` settings.

## 4. Register each project in the dashboard

Open the DockLantern Settings page over HTTPS. Fill out **Register project**. Example for `hello-py`:

| Form field | Value (example; use your actual host path) |
| --- | --- |
| Compose project name | `hello-py` |
| Project directory | `/srv/apps/hello-py` |
| Compose files | `/srv/apps/hello-py/docker-compose.yaml` |
| Services | `web` |
| Environment files | `/srv/apps/hello-py/.env` (if required) |
| Profiles | Leave blank if unused |

The project name and service must match the actual Compose labels. Repeat for `hello-js`, using its own absolute files and `web` service.

Expected message: **Project registered for monitoring.** Registration is not an authorization to deploy or restart containers. The project may already appear in *Discovered services* before registration; they are distinct states.

If registration reports **CSRF validation failed**:
1. Reload the Settings page after any container restart because CSRF tokens are generated per process.
2. Check the browser's Network tab on `POST /settings/projects`. The `Origin` should match the dashboard HTTPS origin, and `Sec-Fetch-Site` should be `same-origin`.
3. Check `GET /settings` response header `Referrer-Policy`. Use `same-origin` for this version. A `no-referrer` response caused Chrome to send `Origin: null` and reject legitimate form submissions.
4. Do **not** disable CSRF validation or relax CSP. See [troubleshooting](troubleshooting.md).

## 5. Create monitoring schedules

For each registered service, choose **Settings → Configure watched service**. Example values:

| Field | hello-py | hello-js |
| --- | --- | --- |
| Project | `hello-py` | `hello-js` |
| Service | `web` | `web` |
| Tagged image | `davidslusser/hello-py:latest` | `davidslusser/hello-js:latest` |
| Platform | `linux/amd64` (check your host) | `linux/amd64` (check your host) |
| Interval (seconds) | `300` | `300` |
| Pause monitoring | Unchecked | Unchecked |

Confirm the local image architecture before setting a platform:

```bash
docker image inspect davidslusser/hello-py:latest --format '{{.Os}}/{{.Architecture}}'
docker image inspect davidslusser/hello-js:latest --format '{{.Os}}/{{.Architecture}}'
```

Save each monitoring policy. Within the configured interval, the Overview's Registry Monitoring section should show `first_seen`, a platform-specific digest, and the check time. Use your actual image names if these are not the images running in your Compose projects.

## 6. Test detection *without* updating a container

Record one running service's identity before publishing an image:

```bash
docker inspect hello-js-web-1 --format 'Container ID: {{.Id}}
Image ID: {{.Image}}
Started at: {{.State.StartedAt}}'
```

Publish a **different** Docker Hub manifest behind the watched tag (e.g. `hello-js:latest`) using its normal CI/CD workflow. **Do not pull or deploy that image on the test host.** After the next scheduled registry check, DockLantern should report `changed` for a digest change.

Wait for one more check with no further image publications. Expected status: `unchanged`. In **Events**, a changed digest should produce one `remote_digest_changed` event; another unchanged check should not duplicate it.

Run the same `docker inspect` command again: the running container ID, image ID, and started-at timestamp should match the original values.

### Observed beta test outcome (2026-10-25)

- `hello-js:latest` was recorded as `first_seen` at 22:14 UTC.
- A new Docker Hub digest resulted in `changed` at 22:19 UTC.
- The next check returned `unchanged` at 22:24 UTC.
- Exactly one `remote_digest_changed` event was recorded; the container ID, image ID, and start time remained unchanged.
- **Known limitation:** the generic registry-change event displayed a blank project/service association (`/ `). That is a separate application attribution issue to address in a later beta.

This confirms OCI digest monitoring and event persistence for the test, **not** that DockLantern updates containers or understands semantic version ordering. The beta remains monitoring-only.

## 7. Notification status is separate

An event can show `pending (0)` when outbound webhook notifications are not configured. This does not indicate that the registry check failed. No webhook delivery test is included in this walkthrough.
