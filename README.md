# vesta

The **interior of the homelab**: a Synology DSM NAS running self-hosted services
in Docker Compose, fronted by **Traefik** (TLS, reverse proxy) with **AdGuard
Home** (DNS + filtering), plus Vaultwarden, JDownloader2, and a whoami probe.

> **vesta** — the Roman goddess of the hearth and the guarded centre of the
> home, whose temple safeguarded Rome's sacred objects: a fitting emblem for the
> box that *hosts and holds*. The network perimeter — the router running
> Tailscale — lives in a separate repository,
> [`janus`](https://github.com/LRNZ09/janus), the two-faced god of the doorway.
> Together they split the homelab into *interior* and *threshold*.

## Important: there is **no Tailscale here**

Remote access is handled entirely by `janus` (the router) acting as a Tailscale
subnet router + exit node. The NAS runs **zero** Tailscale containers. This is
deliberate — see `docs/architecture.md` for why (single NIC + macvlan host
isolation make the router the correct place for Tailscale). Most NAS guides put
Tailscale on the NAS; this homelab specifically does not.

## The shape of it

- **Traefik** owns a **macvlan** address `192.168.50.254` (its own L2 identity
  on the LAN) so it can hold `:80/:443` without colliding with DSM, which keeps
  those ports on the host. It also sits on the shared **`traefik` bridge**
  (`172.50.0.0/24`).
- **AdGuard Home** runs in **`network_mode: host`** and owns `:53` on the **NAS
  host address**. Its admin UI is bound to the `traefik` bridge gateway
  (`172.50.0.1:3080`), so only Traefik can reach it — and Traefik proxies it at
  `adguardhome.admin.lorenzopieri.dev`. Running host-mode keeps DNS up even when
  Traefik is recreated.
- **Bridge backends** (Vaultwarden, JDownloader2, whoami) attach to the
  `traefik` bridge and are reverse-proxied by Traefik.

## Repository layout

```tree
vesta/
├── README.md
├── LICENSE
├── .gitignore
├── .env.example            # shared host UID/GID (copy to .env)
├── docs/
│   ├── architecture.md     # macvlan + host-mode design; why no Tailscale here
│   ├── traefik.md          # networks, Cloudflare DNS-01, routing, dashboards
│   ├── adguardhome.md      # host-mode DNS, rewrites, LAN + tailnet DNS
│   ├── services.md         # how to add a service; per-service notes
│   ├── operations.md       # bring-up order, backups
│   └── disaster-recovery.md
└── services/
    ├── traefik/
    │   ├── compose.yaml     # macvlan (.254) + traefik bridge owner
    │   ├── traefik.yaml     # static config (entrypoints, le/Cloudflare resolver)
    │   ├── .env.example     # CF_DNS_API_TOKEN, CF_ZONE_API_TOKEN, basic-auth slot
    │   ├── config/          # tls, dashboard, adguardhome, lan-only, https-transport
    │   └── le/              # ACME store (acme.json) — gitignored
    ├── adguardhome/
    │   ├── compose.yaml     # network_mode: host; DNS :53
    │   ├── .gitignore       # ignores work/ runtime + the real dnscrypt.yaml
    │   └── conf/            # AdGuardHome.yaml (hash redacted) + dnscrypt.yaml.example
    ├── vaultwarden/         # compose.yaml + .env.example
    ├── jdownloader-2/       # compose.yaml + curated config .gitignores
    └── whoami/              # compose.yaml (routing/TLS probe)
```

## Quick start

Traefik comes up **first** — its compose defines the `traefik` bridge
(`172.50.0.0/24`) and the macvlan, so backends have a network to attach to. The
root `.env` sets the host `UID`/`GID` the containers run as and is passed to every
command with `--env-file .env`. Secrets stay in per-service `.env` files (not
committed). Run from the repo root:

```sh
cp .env.example .env                                     # set UID/GID (host user)
cp services/traefik/.env.example services/traefik/.env   # fill in Cloudflare tokens
docker compose --env-file .env -f services/traefik/compose.yaml up -d     # network owner FIRST

docker compose --env-file .env -f services/adguardhome/compose.yaml up -d # host-mode DNS
cp services/vaultwarden/.env.example services/vaultwarden/.env
docker compose --env-file .env -f services/vaultwarden/compose.yaml up -d # then bridge backends
docker compose --env-file .env -f services/jdownloader-2/compose.yaml up -d
docker compose --env-file .env -f services/whoami/compose.yaml up -d
```

Order matters and `depends_on` does not cross compose files — see
`docs/operations.md`.

## Secrets / before you publish

This repo is **public**. Secrets live in per-service `.env` files (gitignored);
only each `.env.example` (no values) is committed. Required secrets: Cloudflare
API tokens (Traefik DNS-01) and the Vaultwarden admin token. In addition:

- `services/traefik/le/acme.json` (ACME private keys) is gitignored.
- `services/adguardhome/conf/dnscrypt.yaml` (DNSCrypt private keys) is
  gitignored; a redacted `dnscrypt.yaml.example` is committed instead.
- The AdGuard admin password hash in `conf/AdGuardHome.yaml` is redacted.
- `services/adguardhome/work/` (query log, stats, sessions, filters) is
  gitignored.

The only environment specifics committed are an RFC 1918 subnet and the public
service domain (already in Certificate Transparency logs).

## Secret scanning

- Pre-commit hook (`.githooks/pre-commit`) runs `gitleaks` on staged changes.
  Enable per clone: `git config core.hooksPath .githooks` (needs `gitleaks`).
- CI (`.github/workflows/gitleaks.yml`) scans every push and PR.

## License

MIT — see [LICENSE](LICENSE).
