# Deployment Surface

## Dockerfile (multi-stage)

```
Stage 1 (builder):  node:22-alpine → install all deps, compile TypeScript
Stage 2 (runner):   node:22-alpine → copy dist/ + node_modules (prod only)
Expose: 3000
CMD: node dist/main.js
```

Migrations run as a separate one-shot command (`npx drizzle-kit migrate`) in the deploy pipeline before the new container starts — not at runtime inside the app process. The `drizzle/` migration folder is baked into the image so the migrator and the app run identical migration state.

## Environment variables (12-factor)

```
DATABASE_URL          postgres://...
JWT_SECRET            <random 256-bit>
JWT_REFRESH_SECRET    <random 256-bit>
GOOGLE_CLIENT_ID      <OAuth client ID for token verification>
FIREBASE_CREDENTIALS  <base64-encoded service account JSON>
PORT                  3000
NODE_ENV              production
```

No provider-specific SDKs. All configuration through env vars. The same image runs on Fly.io (`fly.toml`), Railway (env panel), Render (`render.yaml`), or a bare VPS (`docker run --env-file .env`).

## Healthcheck

`GET /health` — returns `200 { status: "ok", db: "ok" }` after a `SELECT 1` against Postgres. Used by all PaaS platforms for readiness probes.

## Fresh deploy checklist (any platform)

1. Provision PostgreSQL (managed or self-hosted).
2. Set env vars.
3. Run `npx drizzle-kit migrate` (one-shot container or pre-deploy hook).
4. Deploy the app container.
5. Set up `pg-boss` scheduler: it auto-initializes its schema on first connect.

The only manual step that varies by platform is where env vars are entered.
