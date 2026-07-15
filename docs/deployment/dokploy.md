# Dokploy Deployment

This guide covers deploying Instatic on [Dokploy](https://dokploy.com) using its Docker Compose application type, building from source.

Dokploy takes a single Compose file path per application — it cannot stack the `-f compose.prod.yml -f compose.sqlite.yml` overlay chain the VPS guide uses. `compose.dokploy.yml` and `compose.dokploy-postgres.yml` are self-contained files built specifically for this: each defines the whole stack (`app`, plus `postgres` for the Postgres variant) with `build:` pointed at the repo's `Dockerfile`, `expose:` instead of a published host port, and an external `dokploy-network` attachment so Dokploy's Traefik proxy can route to it.

---

## TL;DR

| Variant | Compose Path | Containers | Persistent volumes |
|---|---|---|---|
| SQLite | `compose.dokploy.yml` | `app` | `data`, `uploads` |
| Postgres | `compose.dokploy-postgres.yml` | `app`, `postgres` | `postgres_data`, `uploads` |

SQLite is the default for single-site installs. Postgres is for multiple simultaneous admin writers or operators who already standardize on Postgres.

## Create the Application

1. In Dokploy, create a new **Application** of type **Docker Compose**.
2. Point it at this repository and the branch you want to deploy.
3. Set **Compose Path** to `compose.dokploy.yml` (SQLite) or `compose.dokploy-postgres.yml` (Postgres).
4. Leave the build type as **Dockerfile** / source build — Dokploy builds the image from the repo on each deploy.

## Environment Variables

Set these in the application's **Environment** panel before the first deploy:

```txt
PUBLIC_ORIGIN=https://cms.example.com
INSTATIC_SECRET_KEY=<output of bun run scripts/generate-secret-key.ts>
```

`PUBLIC_ORIGIN` is required by both compose files (`${PUBLIC_ORIGIN:?...}`) — the deploy fails fast with a clear error if it's missing rather than booting with a broken CSRF check. Dokploy's Traefik terminates TLS and forwards plain HTTP to the container, so the app cannot infer its own public origin from the request URL.

`INSTATIC_SECRET_KEY` is required before adding AI provider credentials, saving plugin secret settings, or enabling TOTP MFA in production. The admin still loads without it.

Postgres deployments additionally require:

```txt
POSTGRES_PASSWORD=<output of: openssl rand -hex 24>
```

`compose.dokploy-postgres.yml` guards this with `${POSTGRES_PASSWORD:?...}` on both the `postgres` and `app` services — the deploy fails rather than starting with a guessable password.

## Domain and TLS

Add your domain in the application's **Domains** tab, pointed at container port `3001`. Dokploy auto-generates the Traefik labels and provisions a Let's Encrypt certificate — do not add a separate TLS proxy (`compose.tls.yml`'s Caddy overlay is for the plain-VPS path and is redundant here; running it alongside Dokploy's Traefik would fight over ports 80/443).

## Persistent Data

| Volume | Mount path | Contents |
|---|---|---|
| `data` (SQLite only) | `/app/data` | SQLite database |
| `postgres_data` (Postgres only) | `/var/lib/postgresql/data` | Postgres data directory |
| `uploads` | `/app/uploads` | Media, fonts, plugins, published artefacts |

These are Docker named volumes managed by Dokploy. Back up both the database and `uploads` — see [backup-restore.md](backup-restore.md).

## Updates

Trigger a redeploy from the Dokploy UI, or configure a Git webhook/auto-deploy on push to the tracked branch. Each deploy rebuilds the image from the current source and re-runs database migrations on boot (`server/index.ts`).

## Troubleshooting

| Symptom | Check |
|---|---|
| Deploy fails immediately with a `Set PUBLIC_ORIGIN...` error | Set `PUBLIC_ORIGIN` in the Environment panel to `https://` plus your Dokploy domain. |
| Deploy fails immediately with a `Set POSTGRES_PASSWORD...` error | Postgres variant only — set `POSTGRES_PASSWORD` in the Environment panel. |
| Admin loads but every save fails with `Forbidden: invalid origin` | `PUBLIC_ORIGIN` doesn't match the domain you're visiting. |
| Domain returns a Traefik 404 / bad gateway | Confirm the Domains tab points at container port `3001` and that the deploy succeeded (container attaches to `dokploy-network` automatically — no manual network config needed). |
| Adding an AI provider credential or enabling TOTP MFA returns 500 | Set `INSTATIC_SECRET_KEY` to the output of `bun run scripts/generate-secret-key.ts`. |

## Related

- [deployment/README.md](README.md) — deployment overview
- [vps.md](vps.md) — the analogous plain Docker Compose path (no Dokploy)
- [backup-restore.md](backup-restore.md) — backup and restore procedures
- `compose.dokploy.yml` — SQLite Dokploy Compose file
- `compose.dokploy-postgres.yml` — Postgres Dokploy Compose file
