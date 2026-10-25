# Troubleshooting DockLantern

Run diagnostic commands on the host in `~/deployments/docklantern`. Start with the logs **before** changing permissions or restarting other services.

## DockLantern keeps restarting with exit code 1

Example symptom: `docklantern-docklantern-1 Restarting (1)`, while `docklantern-docker-socket-proxy-1` stays up.

Read the reason:

```bash
docker compose -f compose.yaml -f compose.discovery.yaml logs --tail=100 docklantern
```

**Observed during the first deployment:** `/run/secrets/admin_password` was unreadable because the local file was owned by the SSH login user, with mode `0600`, while DockLantern ran with `cap_drop: ALL`.

On the standard rootful Docker setup, fix:

```bash
sudo chown root:root secrets/admin_password.txt
sudo chmod 600 secrets/admin_password.txt
docker compose -f compose.yaml -f compose.discovery.yaml up -d --force-recreate --no-deps docklantern
```

Confirm it stays up:

```bash
docker compose -f compose.yaml -f compose.discovery.yaml ps
docker compose -f compose.yaml -f compose.discovery.yaml logs --tail=50 docklantern
```

Other restart causes include missing `.env`, malformed credentials, invalid public origin, inaccessible SQLite storage, or a crashing background worker. Do not assume every restart is a permissions problem. Rootless Docker may need different ownership.

## Traefik says "Router uses a nonexistent certificate resolver"

A `tls.certresolver` label named `letsencrypt` previously referred to a resolver that wasn't configured in the running proxy.

**Fix:** this repository intentionally does **not** supply that label. Traefik must already be configured to provide a valid certificate by another method. Do not add the label unless you've verified the resolver name on the actual proxy.

Read Traefik logs:

```bash
docker logs traefik --since 10m 2>&1 | grep -Ei 'docklantern|router|certificate|error'
```

If your Traefik container has a different name, change `traefik` in the command.

## HTTPS does not load, or certificate is invalid

Check:

```bash
docker network inspect web
docker compose -f compose.yaml -f compose.discovery.yaml ps
curl -Iv https://docklantern.example.com/
```

Replace the example hostname with your real one in the local command. Verify the actual DNS record, certificate, Traefik entry point `websecure`, and that both containers are attached to `web`. An authentication `401` on `/` can be normal; TLS errors, `404` router misses, and `502` backend failures are not.

## Inventory empty or Docker scan failing

```bash
docker compose -f compose.yaml -f compose.discovery.yaml logs --tail=150 docklantern docker-socket-proxy
docker network inspect docklantern_docker-discovery
```

The service needs the *overlay* (`compose.discovery.yaml`), not just `compose.yaml`. Do **not** attach the proxy to a public network, open port `2375`, or add a direct Docker socket mount to DockLantern.

## Docker Hub/GHCR checks fail on an IPv6-only VM

```bash
docker network inspect docklantern-ipv6-egress
docker compose -f compose.yaml -f compose.discovery.yaml logs --tail=100 docklantern
```

Check the host's IPv6 routing and DNS resolver. The presence of the named network does **not** guarantee external connectivity.

## The container inventory shows services but no registered projects

Discovering running containers and registering projects are **different actions**. A new SQLite volume starts without old registrations or policies. You must register projects explicitly, or restore a compatible backup. This beta does not authorize updates just because something was discovered.

## Container says "unhealthy" but is still running

`/readyz` checks worker heartbeats. A temporary registry or Docker error may cause a `503` without exiting the HTTP process. Inspect logs and current state; the normal Docker Compose restart policy doesn't automatically restart a container solely because its healthcheck failed.

## CSRF validation failed when submitting the registration form

This was reproduced during beta testing behind Traefik. On a normal HTML form POST, Chrome submitted `Origin: null` while `Sec-Fetch-Site: same-origin` because the Settings page sent `Referrer-Policy: no-referrer`. DockLantern correctly rejected the origin even though the CSRF token was present.

Use **`Referrer-Policy: same-origin`** in your Traefik security middleware and the application Settings page (fixed in application source for the next beta). After updating the label, validate the Compose configuration and recreate **only** DockLantern. Refresh the Settings page and retry. Check in Chrome Developer Tools that `GET /settings` responds with **`Referrer-Policy: same-origin`** and the POST sends your actual same-origin HTTPS `Origin`.

Do **not** disable CSRF validation, turn off CSP, or permit `Origin: null` in the application.

## Project registered, but files not available after restart

The optional `compose.projects.yaml` overlay must be included in **every** future Compose invocation that recreates DockLantern. Its mounts and `-roots` allowlist are required to validate project registrations. See [monitoring-first-run.md](monitoring-first-run.md). Confirm the directory and file paths match the original Docker Compose labels exactly and are mounted read-only.

## Never use destructive cleanup as a quick fix

**Do not run `docker compose down -v`** unless you've intentionally chosen to delete SQLite state. Also don't delete networks that other containers use. The running Traefik and unrelated applications are managed independently.
