# DockLantern deployment (Docker Compose + Traefik)

**Reproducible, monitoring-only DockLantern beta deployment** for a Linux VPS or cloud VM that **already runs Docker and Traefik**.

This repository holds **deployment configuration**, not DockLantern's source code. The application lives at [docklantern/docklantern](https://github.com/docklantern/docklantern).

> **Beginner note:** These instructions reproduce the DockLantern application stack on a prepared server. They do **not** install Docker, set up Traefik, configure DNS certificates, or build the host's IPv6 routing from scratch. Those are existing host prerequisites. See [Host prerequisites](docs/host-prerequisites.md).

## What you will deploy

- **DockLantern**: web dashboard and registry monitor, pinned to the published **v0.1.0-beta.5** Docker Hub image and its release-workflow-recorded immutable multi-platform index digest. See the [beta.5 prerelease](https://github.com/docklantern/docklantern/releases/tag/v0.1.0-beta.5).
- **Docker socket proxy**: a separate container that permits limited, read-only Docker API requests for container discovery.
- **SQLite database**: durable application state stored in a Docker named volume.
- **Traefik route**: HTTPS access through an **already-running** reverse proxy. This repository does not deploy or restart Traefik.

DockLantern beta.5 remains **monitoring-only**: it does not automatically pull, update, or recreate other services. It retains the beta.3 label-first enrollment for Compose services carrying `com.docklantern.monitor: "true"`, without original Compose source mounts, project registration, or DockLantern configuration changes. Unlabeled services remain opt-out by default. The legacy beta.2 registration path is still supported.

## Files and folders

```text
.
├── compose.yaml                 # DockLantern, auth, healthcheck, TLS route, SQLite volume
├── compose.discovery.yaml       # Optional Docker discovery via a restricted proxy
├── compose.projects.example.yaml # Legacy beta.2/manual registration only (optional)
├── socket-proxy/
│   └── Dockerfile               # Builds the patched socket-proxy image
├── .env.example                 # Safe configuration template; copy to .env
├── .gitignore                   # Excludes secrets, .env, backups, database files
├── .github/workflows/
│   └── validate-compose.yml     # Validates both Compose configurations on PRs
└── docs/
    ├── host-prerequisites.md    # Docker, Traefik, DNS/TLS, external networks
    ├── monitoring-first-run.md # Beta.3 label-based monitoring walkthrough
    ├── operations.md            # Start, stop, upgrade, backup and restore
    ├── beta5-upgrade.md         # Approved-upgrade gates, v3→v5 backup and rollback
    ├── legacy-manual-registration.md # Original beta.2 source-mount registration walkthrough
    └── troubleshooting.md       # Common failures and actual fixes
```

## Existing beta.4.1 installations — read before upgrading

**Do not apply the new Compose image pin to an existing VPS until a separate upgrade/deployment is explicitly approved.** This repository now describes beta.5 as the desired image; it does not mean the running host has been upgraded. Beta.5 introduces **forward-only SQLite migrations v3 → v4 → v5**. An older beta.4.1 binary cannot safely read the migrated v5 database.

Before pulling, recreating, or running **any** container using the new beta.5 Compose definition, take and verify a **consistent SQLite backup using the still-running beta.4.1 image/configuration**, and preserve a copy off-host. A `docker compose run ... -backup` command executed **after** this new Compose file is checked out may start beta.5 and migrate the live database **before taking the backup**.

Follow the [beta.5 upgrade and rollback checklist](docs/beta5-upgrade.md) and [application acceptance plan](https://github.com/docklantern/docklantern/blob/main/docs/deployment/beta5-test-vps-acceptance.md). Merging this deployment PR alone does not authorize a deployment, database migration, target-service restart or operator acceptance.

## Step-by-step first installation

The shell commands run **on your server over SSH**, not on your laptop. Use an account that can run Docker. Lines beginning with `#` are comments, not commands.

### Step 1 — Check the host first

Log into your Linux server, then run:

```bash
docker version
docker compose version
docker ps
docker network ls
```

You should already have a running Traefik container. Check that it uses an HTTPS entry point called `websecure` and is attached to a Docker network called `web`:

```bash
docker network inspect web
docker ps --filter name=traefik
```

This example also expects an **existing, working** Docker network named `docklantern-ipv6-egress`, because the example host uses an IPv6-only VM:

```bash
docker network inspect docklantern-ipv6-egress
```

If either network is missing, **do not proceed**. See [Host prerequisites](docs/host-prerequisites.md). In particular, simply creating an IPv6 network does not guarantee working Internet access.

Set up your own DNS hostname pointing to the existing HTTPS reverse proxy. **Every hostname in this repository is an example**: replace `docklantern.example.com` with a hostname you control. Do not include a real hostname in a committed file.

### Step 2 — Download this repository

Run:

```bash
mkdir -p ~/deployments
cd ~/deployments
git clone https://github.com/docklantern/docklantern-deploy.git docklantern
cd docklantern
```

If a directory named `docklantern` already exists, back it up and inspect it before cloning. Do not overwrite an existing `secrets` directory or delete a Docker volume.

Use `pwd` to confirm that you are in `~/deployments/docklantern`.

### Step 3 — Create your local settings

Copy the provided template:

```bash
cp .env.example .env
chmod 600 .env
nano .env
```

Replace the placeholder hostname with **your real dashboard DNS hostname**, but **only in your local `.env` file**:

```dotenv
DOCKLANTERN_DOMAIN=docklantern.example.com
DOCKLANTERN_ADMIN_USERNAME=admin
```

Replace `docklantern.example.com` before starting the service; it is not a real deployment hostname. The domain must **not** include `https://`, a path, or a trailing slash. To save in Nano, press `Ctrl+O`, `Enter`, then `Ctrl+X`.

The `.env` file is ignored by Git. **Never commit it.** These two values aren't secret credentials, but local settings should stay local to avoid publishing real hostnames.

### Step 4 — Generate a new administrator password

The password lives in a **file-backed Docker secret** rather than in the Compose configuration. Run:

```bash
mkdir -p secrets
chmod 700 secrets
umask 077
openssl rand -base64 36 > secrets/admin_password.txt
sudo chown root:root secrets/admin_password.txt
sudo chmod 600 secrets/admin_password.txt
sudo ls -l secrets/admin_password.txt
```

The last command should display ownership `root root` and permissions `-rw-------`.

**This ownership matters.** DockLantern drops Linux capabilities (`cap_drop: ALL`). On the rootful Docker host used for this deployment, a `0600` secret owned by the normal SSH user caused DockLantern to restart continuously with a permission error. Using `root:root` and `0600` fixed it. On rootless Docker or user-namespace remapping, verify the mapped container UID instead of assuming this ownership works.

To see the password once so you can save it in a password manager:

```bash
sudo cat secrets/admin_password.txt
```

Do not share the password in screenshots, chat messages, issues, or logs. Never commit `secrets/admin_password.txt`.

### Step 5 — Validate configuration before starting anything

```bash
docker compose -f compose.yaml -f compose.discovery.yaml config -q
```

**Expected:** no output and exit status 0. If you see an error, fix it before continuing. The first file defines DockLantern; the second adds the Docker discovery proxy and the IPv6 egress connection.

You can also see the final merged configuration (avoid sharing complete output without reviewing it):

```bash
docker compose -f compose.yaml -f compose.discovery.yaml config
```

### Step 6 — Build, pull, and start

```bash
# Build the patched Docker socket proxy locally.
docker compose -f compose.yaml -f compose.discovery.yaml build docker-socket-proxy

# Download the specified DockLantern beta image.
docker compose -f compose.yaml -f compose.discovery.yaml pull docklantern

# Start only the services managed by this repository.
docker compose -f compose.yaml -f compose.discovery.yaml up -d --no-build
```

`up -d` means "start in the background." It creates the `docker-discovery` internal network and the SQLite volume if they don't exist. It **does not** recreate your existing `web` or IPv6 egress networks.

**Never run `docker compose down -v` during normal operations:** the `-v` option can delete DockLantern's SQLite data volume.

### Step 7 — Confirm both services are healthy

```bash
docker compose -f compose.yaml -f compose.discovery.yaml ps
docker compose -f compose.yaml -f compose.discovery.yaml logs --tail=100 docklantern docker-socket-proxy
docker ps
```

You should see two DockLantern-project services: `docklantern` and `docker-socket-proxy`. DockLantern should remain `Up` (ideally `healthy`), **not** `Restarting (1)`. Other preexisting containers are unaffected.

To test your site, replace the placeholder in this command **locally**:

```bash
curl -Iv https://docklantern.example.com/
curl -i https://docklantern.example.com/readyz
```

Verify the certificate is valid. A `401 Unauthorized` from the login-protected home page is normal before authentication; an HTTP `200` from `/readyz` means the application reports ready. A `503` indicates it is not ready yet; inspect worker logs.

### Step 8 — Sign in and confirm inventory

Open `https://docklantern.example.com/` in your browser, using your actual hostname. Log in with the username from `.env` (default `admin`) and the new password saved in Step 4.

Confirm you can see running Docker containers in the inventory. They may be **discovered but unregistered**. A newly initialized database does not magically restore registrations or history from an earlier Docker volume.

The deployment repository now pins **beta.5**, which retains label-first and GUI monitoring enrollment and the image-first dashboard, and adds watched-tag override/reset and background Check Now. Existing source registrations and monitoring history remain supported; follow the [legacy registration guide](docs/legacy-manual-registration.md) only when maintaining those registrations.

With **beta.5** installed, **monitoring-only** enrollment needs no source files or registration: add `com.docklantern.monitor: "true"` to the *target application's* Compose service, redeploy only that service, and let the existing discovery scanner pick it up. No DockLantern Compose changes or DockLantern restart. See [beta.3 label-first walkthrough](docs/monitoring-first-run.md).

### Step 9 — Optional legacy source registration (beta.2 compatibility)

**Only do this if you are using beta.2's manual source-registration workflow or maintaining existing legacy source registrations.** Beta.3 label-based monitoring does not need this overlay. Keep an existing overlay when upgrading until you have separately validated what other legacy registrations require. The file is local so no VPS directory paths are committed.

```bash
cp compose.projects.example.yaml compose.projects.yaml
nano .env
```

In your local `.env`, update `HELLO_PY_PROJECT_DIR` and `HELLO_JS_PROJECT_DIR` to the actual **absolute** directories for those projects. They must exist on the server and contain the original Compose file and any necessary environment files. Review each directory for secrets before mounting it read-only. See [the legacy manual walkthrough](docs/legacy-manual-registration.md).

Then validate and recreate **only DockLantern**, including the third overlay:

```bash
docker compose -f compose.yaml -f compose.discovery.yaml -f compose.projects.yaml config -q
docker compose -f compose.yaml -f compose.discovery.yaml -f compose.projects.yaml up -d --no-deps docklantern
docker compose -f compose.yaml -f compose.discovery.yaml -f compose.projects.yaml ps
```

**Important:** From this point on, include `-f compose.projects.yaml` in future `docker compose` commands that recreate the DockLantern application; omitting it may remove the read-only mounts and `-roots` allowlist. The original container inventory will continue to work, but registrations and subsequent file validation may not.

This project's Traefik configuration uses `Referrer-Policy: same-origin`, not `no-referrer`. The earlier value caused Chrome to send `Origin: null` on HTML form submissions, which DockLantern correctly rejected as a CSRF failure. The browser workaround was confirmed on the test deployment and is being incorporated in the application code.

## What persists (and what doesn't)

The local `.env` and `secrets/admin_password.txt` are deliberately untracked. The database is stored in the Docker named volume `docklantern_docklantern_data` (Compose project name `docklantern`). It contains registrations and monitoring history. **Cloning this repository again does not restore that data.** Use the [backup procedure](docs/operations.md).

The `web` network, IPv6 egress network, DNS records, Traefik configuration, and TLS certificates are managed **outside** this repository. Document their location before attempting a full-VM disaster recovery.

## Next steps and useful docs

- [Host prerequisites and network considerations](docs/host-prerequisites.md)
- [Beta.3: opt in a Compose app with one Docker service label](docs/monitoring-first-run.md) — included in the published beta.3 release
- [Beta.2: source-based registration and legacy policy management](docs/legacy-manual-registration.md)
- [Beta.5 upgrade, migration and rollback safety](docs/beta5-upgrade.md)
- [Normal operations, upgrading, and backing up SQLite](docs/operations.md)
- [Troubleshooting the actual issues seen during setup](docs/troubleshooting.md)

## Security and licensing

The app never mounts the Docker socket directly; only the isolated helper does. The socket proxy rejects mutation methods and restricts endpoint access, but it still holds a host-privileged socket, and permitted Docker reads can reveal sensitive metadata. **Do not publish socket-proxy port 2375, connect unrelated containers to its private network, or expose DockLantern's plain HTTP port directly.**

The dashboard uses HTTP Basic authentication and must be accessed through trusted HTTPS. Keep Docker, Traefik, images, and secrets patched and audited. Pin tested release versions (and ideally digests); avoid `latest` for production-like beta testing.

This repository is licensed under the existing [MIT License](LICENSE). The application code may have its own license.
