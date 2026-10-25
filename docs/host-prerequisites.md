# Host prerequisites (read before installation)

This repository deploys **DockLantern only**. It assumes an existing Linux host, Docker, working reverse proxy, DNS/TLS setup, and (for this sample) outbound IPv6 network. **It cannot recreate an entire VM by itself.**

## 1. Linux host and Docker

The documented host is a small Ubuntu-based, IPv6-only cloud VM using **rootful** Docker Engine and the Docker Compose v2 plugin.

From an SSH shell:

```bash
docker version
docker compose version
docker info
```

If Docker isn't installed yet, follow the [official Docker Engine Ubuntu installation instructions](https://docs.docker.com/engine/install/ubuntu/). Use the [official Docker Compose plugin instructions](https://docs.docker.com/compose/install/linux/). Be aware that membership in the `docker` group is effectively host-administrator access.

Other tools used in the README: `git` to clone the repository, `openssl` to generate a password, `nano` (or another editor), and `curl` for HTTP checks.

## 2. Existing Traefik proxy (not created here)

The Compose files expect:

- An existing container running Traefik version 3 (the working reference used `traefik:v3.6`).
- An existing **external Docker network** named `web`.
- Traefik and DockLantern attached to `web`.
- Traefik's HTTPS entry point named `websecure`.
- A valid TLS certificate supplied by your existing Traefik and DNS/CDN configuration.

Check:

```bash
docker ps --filter name=traefik
docker network inspect web
docker inspect traefik --format '{{json .NetworkSettings.Networks}}'
```

If your Traefik container has a different name, substitute it in the last command. Read the output before sharing it; inspect output can reveal internal addresses and metadata.

**Do not copy a guessed TLS certificate resolver name** into the Compose labels. The initial deployment failed because the router requested a resolver named `letsencrypt` that did not exist in the running Traefik configuration. This deployment deliberately omits a `tls.certresolver` label. That is only valid if your existing Traefik configuration already supplies a working certificate for your chosen hostname.

## 3. DNS and HTTPS

Choose a hostname you own, such as `docklantern.example.com` (an example only). Configure its DNS using your provider so it points to the existing Traefik service.

If your DNS provider proxies HTTPS (for example, Cloudflare), configure DNS records and TLS modes appropriately, using a valid certificate at the origin where required. Avoid issuing a public Basic-auth challenge over plaintext HTTP.

Before entering admin credentials, confirm the certificate is trusted and matches the **real** hostname:

```bash
curl -Iv https://docklantern.example.com/
```

Change the hostname locally; don't paste it back into this public repository.

## 4. Docker networks and IPv6-only hosts

This Compose example expects two **existing external** networks plus one Compose-owned network:

| Network | Purpose | Created by this Compose stack? |
| --- | --- | --- |
| `web` | Existing Traefik ↔ DockLantern communication | **No** |
| `docklantern-ipv6-egress` | Docker registry/API egress on an IPv6-only host | **No** |
| `docker-discovery` | Internal-only DockLantern ↔ socket proxy traffic | **Yes** |

Check:

```bash
docker network inspect web
docker network inspect docklantern-ipv6-egress
docker network ls
```

A Docker IPv6 network requires **working subnet assignment, default routing, firewall rules, and upstream connectivity**. Merely running `docker network create --ipv6 docklantern-ipv6-egress` may create a network with *no functional Internet route*. If the external network is missing, stop and restore the host's tested configuration. If you have a normal dual-stack host, review the network setup and adapt the overlay rather than creating an unnecessary IPv6 network.

The socket proxy attaches **only** to the private `docker-discovery` network. Do not put it on `web` or give it any published host ports.

## 5. What to back up separately

For full disaster recovery, preserve the configuration of **all** external dependencies in their own infrastructure repositories or secure backups:

- Traefik's Compose/static/dynamic configuration, certificates and TLS resolver setup.
- DNS/CDN records and required certificate or API secrets (never in this repository).
- Host Docker daemon IPv6 configuration, provisioned network settings, firewall rules and routing.
- DockLantern's untracked `.env`, root-owned password file, and a consistent off-host SQLite backup.

See [operations.md](operations.md) for backup and restore.
