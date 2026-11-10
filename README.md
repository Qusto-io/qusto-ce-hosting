# Qusto CE: self-hosted install

Run **Qusto Community Edition** on your own server with Docker Compose.

Qusto CE is a fork of [Plausible Analytics](https://github.com/plausible/analytics)
under the GNU AGPL v3. Source: [Qusto-io/qusto-ce](https://github.com/Qusto-io/qusto-ce).

> **Status: pre-release.** The first public image, `ghcr.io/qusto-io/qusto-ce-community:v1.0.0`,
> is published once its release commit has run in Qusto's own production for 7 days.
> Until then, [build the image from source](#before-v100-build-from-source).

## What you get, and what you don't

| You get | Not included |
|---|---|
| Privacy-first traffic analytics: sources, pages, locations, devices | Funnels |
| Goals and custom properties | Store orders, revenue and products |
| The Stats API | The Coverage figure, customer LTV, attribution |
| No tracking cookies; IP masking for the visitor hash by default | Shopify and WooCommerce connectors |

The right-hand column is managed Qusto: separate services, not part of CE.
Self-hosting is **community supported**. Ask in this repository's issues.

Whether you need consent depends on your market and setup. See `CONSENT_REQUIRED`
in [.env.example](.env.example).

## Requirements

- A Linux server with Docker and the Compose plugin: 2 GB RAM minimum, 4 GB recommended.
- A CPU that supports SSE 4.2 (x86-64) or NEON (arm64). ClickHouse needs it.
- A domain name pointing at the server.

## Install

```sh
git clone https://github.com/Qusto-io/qusto-ce-hosting
cd qusto-ce-hosting
cp .env.example .env
# Edit .env: BASE_URL, and SECRET_KEY_BASE from `openssl rand -base64 48`
```

Choose how the site is served:

- **Built-in HTTPS:** set `HTTP_PORT=80` and `HTTPS_PORT=443` in `.env` and publish both ports
  in a `compose.override.yml`. The app obtains a Let's Encrypt certificate itself.
- **Your own reverse proxy:** `cp compose.override.example.yml compose.override.yml`, which
  publishes the app on `127.0.0.1:8000`. Then use [reverse-proxy/](reverse-proxy/)
  (Caddy or nginx).

```sh
docker compose up -d
```

Open `BASE_URL` and register the first account. Then set `DISABLE_REGISTRATION=invite_only`
(or `true`) and run `docker compose up -d` again.

## Upgrade

1. Read the release notes: <https://github.com/Qusto-io/qusto-ce/releases>.
2. Back up (below).
3. Set the new tag in `QUSTO_CE_IMAGE` (`.env`), or pull the latest version of this repository.
4. Run:

```sh
docker compose pull
docker compose up -d   # database migrations run on start
```

## Verify the image

Releases are signed with keyless cosign from Qusto's GitHub Actions:

```sh
cosign verify ghcr.io/qusto-io/qusto-ce-community:v1.0.0 \
  --certificate-identity-regexp 'https://github.com/Qusto-io/qusto-ce/' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

## Back up and restore

The data lives in three Docker volumes: `db-data` (Postgres: accounts and sites),
`event-data` (ClickHouse: events) and `plausible-data`. The simplest consistent backup
is to stop the stack and archive the volumes:

```sh
docker compose down
for v in db-data event-data plausible-data; do
  docker run --rm -v "$(basename "$PWD")_$v":/data -v "$PWD":/backup alpine \
    tar czf "/backup/$v-$(date +%F).tar.gz" -C /data .
done
docker compose up -d
```

To restore, stop the stack, extract each archive into its (empty) volume the same way
(`tar xzf`), and start again. Keep `.env`: `SECRET_KEY_BASE` must match the backup.

## Before v1.0.0: build from source

```sh
git clone https://github.com/Qusto-io/qusto-ce
docker build -t qusto-ce:local qusto-ce   # MIX_ENV=ce by default
echo 'QUSTO_CE_IMAGE=qusto-ce:local' >> .env
docker compose up -d
```

## How this repository is tested

[`.github/workflows/smoke.yml`](.github/workflows/smoke.yml) runs on every change and weekly.
It boots this compose file with the CE image, creates an account and a site inside the
running app, sends pageviews, and passes only when one is **stored in ClickHouse**.
(A new site reaches the ingestion cache within 30 seconds; events sent before that are
accepted but dropped.)

## Licence

MIT. Based on [plausible/community-edition](https://github.com/plausible/community-edition);
see [NOTICE](NOTICE). The software it installs is AGPL-3.0.
