# Dokploy Deployment

This guide covers a single-site SQLite install of Instatic on [Dokploy](https://dokploy.com). The source of truth for the stack is `compose.dokploy.yml`.

Dokploy runs Traefik on ports 80 and 443 and deploys one Compose file. `compose.dokploy.yml` is the VPS stack (`compose.prod.yml` + `compose.sqlite.yml` + `compose.build.yml`) collapsed into that one file: the app image is built from the repo `Dockerfile`, SQLite lives on a named volume, and TLS is Traefik's job. Do not layer `compose.tls.yml` on this install. Caddy would bind 80/443, which Traefik already owns.

The root `docker-compose.yml` is not this stack. It is the local-dev Postgres service that `bun run dev` starts. Leave it alone.

---

## TL;DR

| Item | Value |
|---|---|
| Compose path in Dokploy | `./compose.dokploy.yml` |
| Service | `app` |
| Container port | `3001` |
| Database | `sqlite:/app/data/cms.db` on the `data` volume |
| Media | `/app/uploads` on the `uploads` volume |
| Public TLS | Dokploy Domains tab (Let's Encrypt). No Traefik labels in the compose file |
| Health | `GET /health` → `{"status":"ok",...}` |

Set these in the Dokploy Environment tab. Dokploy writes them to a `.env` beside the compose file and does not inject that file into the container; `compose.dokploy.yml` references each name.

```txt
INSTATIC_SECRET_KEY=<output of bun run scripts/generate-secret-key.ts>
PUBLIC_ORIGIN=https://lavanderiasantiago.com,https://www.lavanderiasantiago.com
TRUSTED_PROXY_CIDRS=172.16.0.0/12
```

`PUBLIC_ORIGIN` is required. Traefik terminates TLS and forwards plain HTTP to `app:3001`, so the CSRF check in `server/config.ts` cannot infer the public origin from the request URL.

## Prerequisites

- Dokploy is already installed on the VPS. Traefik listens on 80 and 443.
- The Git provider in Dokploy can read `MCFernando96/Instatic`.
- The deploy command includes `--build`. This file builds `instatic:dokploy` from the `Dockerfile`. It does not pull `ghcr.io/corebunch/instatic` (that image is `linux/amd64` only and does not contain fork commits).
- The VPS has enough RAM for the image build. `bun run build` inside the Dockerfile is comfortable at 4 GB.
- DNS for the public host is editable in Cloudflare.

## What the compose file does

- Builds the `app` service from the repo `Dockerfile` and tags it `instatic:dokploy`.
- Sets `DATABASE_URL=sqlite:/app/data/cms.db`, `UPLOADS_DIR=/app/uploads`, `STATIC_DIR=/app/dist`, and `PORT=3001`.
- Mounts named volumes `data` (`/app/data`, the SQLite file) and `uploads` (`/app/uploads`: media, fonts, plugin packs, and published HTML).
- Joins the external network `dokploy-network` so Traefik can reach the container.
- Publishes container port `3001` on a random host port (`ports: ["3001"]`). The public door is the domain, not `:3001`.
- Leaves the domain out of the file. The Domains tab injects the Traefik router. Labels in the file collide with that router.
- Does not set `container_name`. Dokploy uses the project name for logs and metrics.

`INSTATIC_SECRET_KEY` is the AES key for reversible server secrets (AI provider credentials, plugin secret settings, TOTP MFA seeds). Generate it locally and paste it only into Dokploy:

```sh
bun run scripts/generate-secret-key.ts
```

The admin UI loads without the key. Saving those secrets fails until the key is set. Losing or rotating the key makes existing ciphertext unreadable.

`TRUSTED_PROXY_CIDRS=172.16.0.0/12` trusts `X-Forwarded-For` from the Docker bridge range where Traefik connects. That value is client-IP attribution for audit logs and rate-limit keys. It is not the CSRF check. Do not set `0.0.0.0/0`.

## Cloudflare

Point the domain at the VPS before the first deploy so Let's Encrypt can complete its HTTP-01 challenge.

1. `A` record for `@` → the VPS public IP.
2. `A` or `CNAME` for `www` → the same origin.
3. Leave both records DNS-only (grey cloud) until `https://lavanderiasantiago.com/health` returns 200. An orange-cloud proxy in front of a certificate that does not exist yet makes the HTTP-01 challenge fail.
4. After the certificate is issued, the proxy can be turned on. Set SSL/TLS mode to **Full (strict)**.
5. Add a redirect rule from `www.lavanderiasantiago.com` to `https://lavanderiasantiago.com` so the canonical host is the apex. Keep both origins in `PUBLIC_ORIGIN` so a request that still reaches the origin on either host passes the CSRF check.

## Dokploy

1. Create a project, then a Compose service of type Docker Compose.
2. Source: GitHub repository `MCFernando96/Instatic`, branch `main`.
3. Compose path: `./compose.dokploy.yml`.
4. Environment tab: `INSTATIC_SECRET_KEY`, `PUBLIC_ORIGIN`, and `TRUSTED_PROXY_CIDRS` as in the TL;DR. Do not commit those values.
5. Confirm the deploy command includes `--build`.
6. Domains tab: service `app`, container port `3001`, host `lavanderiasantiago.com`, HTTPS, certificate type Let's Encrypt. Add `www.lavanderiasantiago.com` the same way when both hosts must reach the origin before the Cloudflare redirect.
7. Deploy. The first build runs `bun run build` inside the image.
8. Open `https://lavanderiasantiago.com/admin` immediately. The first visit creates the site and the admin account.

## Verify

```sh
curl -fsS https://lavanderiasantiago.com/health
```

Expected body:

```json
{"status":"ok","ts":1234567890}
```

Then sign in at `/admin` and upload a file. Redeploy. The upload and the SQLite database are still there, because `data` and `uploads` are named volumes.

## Data safety

`docker compose down` stops the container and keeps named volumes.

`docker compose down -v` deletes named volumes. For this file that deletes `cms.db` and every upload. Use it only to wipe the install on purpose.

Backups are covered in [backup-restore.md](backup-restore.md). Copy the SQLite file from the `data` volume and the tree from the `uploads` volume. The Postgres commands in that guide do not apply to this install.

## Related

- [deployment/README.md](README.md) — deployment overview
- [vps.md](vps.md) — Compose on a VPS without Dokploy, including bundled Postgres and Caddy
- [docker-image.md](docker-image.md) — image contract (`PORT`, `DATABASE_URL`, `UPLOADS_DIR`, `PUBLIC_ORIGIN`)
- [backup-restore.md](backup-restore.md) — backup and restore
- `compose.dokploy.yml` — the Dokploy stack
- `Dockerfile` — the image this file builds
- `server/config.ts` — `PUBLIC_ORIGIN` and `TRUSTED_PROXY_CIDRS`
