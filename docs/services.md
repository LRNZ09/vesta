# Services

The stack currently runs: **Traefik**, **AdGuard Home**, **Vaultwarden**,
**JDownloader2**, and **whoami**. Traefik and AdGuard have their own docs
(`traefik.md`, `adguardhome.md`); the rest are covered here.

## Adding a new service behind Traefik

1. Attach the container to the external `traefik` bridge: `networks: [traefik]`.
2. Add the routing labels (see `traefik.md`), choosing a hostname under
   `*.admin.lorenzopieri.dev` (covered by the wildcard cert automatically).
3. Set `loadbalancer.server.port` to the container's internal port.
4. `docker compose -f services/<svc>/compose.yaml up -d`.

No DNS change is needed per service: the wildcard rewrite already points every
`*.admin.lorenzopieri.dev` name at `.254`. All proxied services are LAN-only via
the global `lan-only` middleware.

## Per-service notes

### Vaultwarden (`vaultwarden.admin.lorenzopieri.dev`)

- Image `vaultwarden/server:1.36.0-alpine`; internal port `80`.
- The admin token is a **secret**: the compose reads `${VAULTWARDEN_ADMIN_TOKEN}`
  from `.env` and passes it to the container as `ADMIN_TOKEN` (which guards
  `/admin`). Only `.env.example` is committed.
- `DOMAIN` must be the full `https://vaultwarden...` URL or WebAuthn/2FA breaks
  behind the proxy.
- `SIGNUPS_ALLOWED=false` — create your account first, then keep it off.
- The `./data` volume holds the database and secrets and is gitignored.

### JDownloader2 (`jdownloader-2.admin.lorenzopieri.dev`)

- Image `jlesage/jdownloader-2`; web UI (noVNC) on port `5800`, served over
  **HTTPS** (`SECURE_CONNECTION=1`). The router therefore uses
  `loadbalancer.server.scheme: https` plus the `insecure-https-transport@file`
  transport (self-signed backend).
- `WEB_AUTHENTICATION=1` gates the UI; raw VNC is disabled
  (`VNC_LISTENING_PORT=-1`).
- Link your MyJDownloader account in the **web UI** at first run — its
  credentials are not env vars and are not stored in this repo.
- `./config` and `./output` are bind-mounted; per-directory `.gitignore` files
  version a curated set of settings JSONs and exclude downloads/logs/runtime.

### whoami (`whoami.admin.lorenzopieri.dev`)

- `traefik/whoami` echo server — a connectivity/routing probe. Hit it to confirm
  DNS, TLS, and Traefik routing are healthy end to end.
