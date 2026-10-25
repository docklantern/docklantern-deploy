# DockLantern deployment (Docker Compose + Traefik)

**Reproducible, monitoring-only DockLantern beta deployment** for a Linux VPS or cloud VM that **already runs Docker and Traefik**.

This repository holds **deployment configuration**, not DockLantern's source code. The application lives at [docklantern/docklantern](https://github.com/docklantern/docklantern).

> **Beginner note:** These instructions reproduce the DockLantern application stack on a prepared server. They do **not** install Docker, set up Traefik, configure DNS certificates, or build the host's IPv6 routing from scratch. Those are existing host prerequisites. See [Host prerequisites](docs/host-prerequisites.md).

## What you will deploy

- **DockLantern**: web dashboard and registry monitor, using the pinned `davidslusser/docklantern:v0.1.0-beta.2` image.
- **Docker socket proxy**: a separate container that permits limited, read-only Docker API requests for container discovery.
- **SQLite database**: durable application state stored in a Docker named volume.
- **Traefik route**: HTTPS access through an **already-running** reverse proxy. This repository does not deploy or restart Traefik.

This beta is **monitoring-only**: it must not automatically pull, update, or recreate other services. Discovered containers do not become registered projects automatically.

## Files and folders

```text
.
├── compose.yaml                 # DockLantern, auth, healthcheck, TLS route, SQLite volume
├── compose.discovery.yaml       # Optional Docker discovery via a restricted proxy
├── compose.projects.example.yaml # Copy locally for read-only Compose source access
├── socket-proxy/
│   └── Dockerfile               # Builds the patched socket-proxy image
├── .env.example                 # Safe configuration template; copy to .env
├── .gitignore                   # Excludes secrets, .env, backups, database files
├── .github/workflows/
│   └── validate-compose.yml     # Validates both Compose configurations on PRs
└── docs/
    ├── host-prerequisites.md    # Docker, Traefik, DNS/TLS, external networks
    ├── monitoring-first-run.md # Register Compose apps and verify digest monitoring
    ├── operations.md            # Start, stop, upgrade, backup and restore
    └── troubleshooting.md       # Common failures and actual fixes
```

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

Confirm you can see running Docker containers in the inventory. They may be **discovered but unregistered**. Registration is an explicit step and stored in SQLite; a fresh database will **not** automatically restore previously registered projects.

At this point the monitoring-only beta deployment is ready. **Container discovery is not Compose project registration.** To register and monitor an existing Compose project, you must also mount its Compose files into DockLantern and allowlist the same absolute source paths. See [Step-by-step registration and monitoring](docs/monitoring-first-run.md).

### Step 9 — Optionally allow existing Compose projects to be registered

Do this only after the baseline dashboard works. The project registration overlay is **optional** and kept local so no VPS directory paths are committed.

```bash
cp compose.projects.example.yaml compose.projects.yaml
nano .env
```

In your local `.env`, update `HELLO_PY_PROJECT_DIR` and `HELLO_JS_PROJECT_DIR` to the actual **absolute** directories for those projects. They must exist on the server and contain the original Compose file and any necessary environment files. Review each directory for secrets before mounting it read-only. See [the complete beginner walkthrough](docs/monitoring-first-run.md).

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
- [Register two Compose apps and validate scheduled registry monitoring](docs/monitoring-first-run.md)
- [Normal operations, upgrading, and backing up SQLite](docs/operations.md)
- [Troubleshooting the actual issues seen during setup](docs/troubleshooting.md)

## Security and licensing

The app never mounts the Docker socket directly; only the isolated helper does. The socket proxy rejects mutation methods and restricts endpoint access, but it still holds a host-privileged socket, and permitted Docker reads can reveal sensitive metadata. **Do not publish socket-proxy port 2375, connect unrelated containers to its private network, or expose DockLantern's plain HTTP port directly.**

The dashboard uses HTTP Basic authentication and must be accessed through trusted HTTPS. Keep Docker, Traefik, images, and secrets patched and audited. Pin tested release versions (and ideally digests); avoid `latest` for production-like beta testing.

This repository is licensed under the existing [MIT License](LICENSE). The application code may have its own license.
