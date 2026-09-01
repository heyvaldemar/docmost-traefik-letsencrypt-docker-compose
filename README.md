# Docmost + Traefik + Let's Encrypt — Docker Compose

[![Deployment Verification](https://github.com/heyvaldemar/docmost-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml/badge.svg?branch=main)](https://github.com/heyvaldemar/docmost-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

This repository deploys **Docmost** — an open-source collaborative wiki and documentation platform (a self-hosted Confluence/Notion alternative) — behind **Traefik** with automatic **Let's Encrypt TLS**, backed by **PostgreSQL 16** and **Redis**, with scheduled **backups** (database + uploaded files) and companion **restore scripts**.

📙 Full narrative installation guide on the blog: [heyvaldemar.com/install-docmost-using-docker-compose/](https://www.heyvaldemar.com/install-docmost-using-docker-compose/).

## Getting started

```bash
# 1. Clone
git clone https://github.com/heyvaldemar/docmost-traefik-letsencrypt-docker-compose
cd docmost-traefik-letsencrypt-docker-compose

# 2. Create the two Docker networks the stack expects
docker network create traefik-network
docker network create docmost-network

# 3. Copy the environment template and fill in required values
cp .env.example .env
$EDITOR .env
# ^ Required: DOCMOST_DB_PASSWORD, DOCMOST_SECRET, DOCMOST_HOSTNAME,
#   DOCMOST_URL, TRAEFIK_HOSTNAME, TRAEFIK_ACME_EMAIL, TRAEFIK_BASIC_AUTH.

# 4. Deploy
docker compose -f docmost-traefik-letsencrypt-docker-compose.yml -p docmost up -d
```

Docmost runs its migrations on first start; within a minute `https://${DOCMOST_HOSTNAME}` serves the setup page. **The first account registered becomes the workspace owner** — open it right after deploy.

### What success looks like

```bash
docker compose -f docmost-traefik-letsencrypt-docker-compose.yml -p docmost ps
curl -fskL -o /dev/null -w "%{http_code}\n" "https://${DOCMOST_HOSTNAME}/"
```

### Common first-deploy issues

- **Container restarts with an env-validation error.** The log names the variable — `MAIL_DRIVER` accepts only `smtp` or `postmark`, and `DOCMOST_SECRET`/`DOCMOST_URL` must be set.
- **Cert issuance fails.** DNS hasn't propagated or port 80 isn't reachable from the internet.
- **Networks not found.** Step 2 was skipped.
- **Invites don't arrive.** SMTP is unconfigured by default — set the `DOCMOST_SMTP_*` variables, or copy invite links from the UI.

## Supply chain trust

Four images — [`traefik`](https://hub.docker.com/_/traefik), [`docmost/docmost`](https://hub.docker.com/r/docmost/docmost), [`postgres`](https://hub.docker.com/_/postgres), [`redis`](https://hub.docker.com/_/redis) — pinned to `tag@sha256:<digest>` as interpolation defaults in the compose `x-images` block. `git pull` alone delivers the tested combination; an `*_IMAGE_TAG` variable in `.env` overrides deliberately.

The weekly `check-pin-freshness` CI job re-resolves each pin against its registry and compares the pinned Docmost and Traefik versions against the latest upstream releases. GitHub Actions are pinned by commit SHA; Dependabot keeps those fresh.

## Production checklist

- [ ] **Register the owner account immediately after deploy.**
- [ ] **Strong secrets** — the database password and the 64-character app secret; regenerate the Traefik dashboard hash.
- [ ] **Configure SMTP** for invites and password resets.
- [ ] **Host-mount the backup volumes** for disaster recovery.
- [ ] **Docmost is pre-1.0** — read release notes before bumping the pin; back up before upgrades.

## Backups and restore

The `backups` container runs a `pg_dump | gzip` + `tar.gz`-of-uploads → prune → sleep loop (defaults: 30-minute warm-up, 24-hour interval, 7-day retention). Restore with the interactive scripts (`chmod +x *.sh` once): `./docmost-restore-database.sh`, then `./docmost-restore-application-data.sh`.

## Testing

The [Deployment Verification](https://github.com/heyvaldemar/docmost-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml?query=branch%3Amain) workflow runs on every push, pull request, and every Monday at 06:00 UTC: shellcheck + actionlint, Trivy scans of all four pinned images, the weekly freshness check, and a deploy-and-test job that boots the stack with ephemeral credentials and requires the UI to answer through Traefik after migrations complete.

## Security Notes

- Credentials are read from `.env` at deploy time; `.env` is gitignored and compose fails fast on missing required variables.
- **Pre-rotation advisory.** Releases before v1.0.0 (2026-09-01) shipped a tracked `.env` with a generated-looking database password, app secret, and SMTP relay credentials. Rotate them all if your deployment reused them (changing `DOCMOST_SECRET` logs everyone out).
- PostgreSQL and Redis listen only on the internal network.

---

## About the maintainer

<div align="center">

**Maintained by [Vladimir Mikhalev](https://github.com/heyvaldemar)** — Docker Captain · IBM Champion · AWS Community Builder

[YouTube](https://www.youtube.com/channel/UCf85kQ0u1sYTTTyKVpxrlyQ?sub_confirmation=1) · [Blog](https://heyvaldemar.com) · [LinkedIn](https://www.linkedin.com/in/heyvaldemar/)

</div>
