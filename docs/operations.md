# Operations

## Bring-up order

`depends_on` does not work across separate compose files, so order is manual:

1. **Traefik** (`services/traefik/compose.yaml`) — creates the `traefik` bridge
   (`172.50.0.0/24`) and the macvlan, and claims `.254`.
2. **AdGuard** (`services/adguardhome/compose.yaml`) — start it after Traefik so
   the `traefik` bridge gateway (`172.50.0.1`) exists for its admin-UI bind.
3. The **bridge backends** (Vaultwarden, JDownloader2, whoami), which attach to
   the `traefik` network as `external`.

Every command takes `--env-file .env` so the containers pick up the shared host
`UID`/`GID` from the repo-root `.env` (compose does not auto-load it for a
`-f services/<svc>/compose.yaml` invocation). Run these from the repo root:

```sh
docker compose --env-file .env -f services/traefik/compose.yaml up -d
docker compose --env-file .env -f services/adguardhome/compose.yaml up -d
docker compose --env-file .env -f services/vaultwarden/compose.yaml up -d
docker compose --env-file .env -f services/jdownloader-2/compose.yaml up -d
docker compose --env-file .env -f services/whoami/compose.yaml up -d
```

## AdGuard is decoupled from Traefik (no restart cascade)

Because AdGuard runs in **host mode** (not inside Traefik's network namespace),
recreating Traefik — image update, `down`/`up`, config change — does **not** take
DNS down. This is the main reason for the host-mode choice; there is no
"restart AdGuard after Traefik" habit to remember.

The one lifecycle dependency is the AdGuard **dashboard**: its UI binds to the
`traefik` bridge gateway `172.50.0.1:3080`. That gateway exists as long as the
`traefik` bridge does (it persists while any backend is attached). If you ever
tear the bridge down entirely and recreate it, restart AdGuard so it can re-bind:

```sh
docker compose --env-file .env -f services/adguardhome/compose.yaml restart
```

## Updates

- Pin meaningful tags and update deliberately:
  `docker compose -f services/<svc>/compose.yaml pull && ... up -d`.
- Recreating Traefik is safe for DNS (see above), but still verify the wildcard
  cert and dashboards afterward.

## Backups

State lives in **bind-mounted directories under `services/`**, not named volumes:

- **AdGuard:** `services/adguardhome/conf/` (config) and
  `services/adguardhome/work/` (query log, stats, filters). `conf/` is the one
  worth keeping.
- **Vaultwarden:** `services/vaultwarden/data/` — the database and secrets. This
  is the **highest-value** backup; see `disaster-recovery.md`.
- **JDownloader2:** its `config/` directory.
- **Traefik ACME:** `services/traefik/le/acme.json` holds issued certs + private
  keys. Back it up (encrypted) or accept that Traefik re-issues via DNS-01 on a
  rebuild. Never commit it.

Back these up with Synology Hyper Backup over the paths, or e.g.
`tar czf adguard-conf.tgz -C services/adguardhome/conf .`.
