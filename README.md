# vesta

The **interior of the homelab**: a Synology DSM NAS running self-hosted services
in Docker Compose, fronted by **Traefik** (TLS, reverse proxy) with **AdGuard
Home** (DNS + filtering), plus Vaultwarden, Jellyfin, JDownloader2, and
monitoring.

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
  those ports on the host.
- **AdGuard Home** shares Traefik's network namespace, so it lives on the **same
  `192.168.50.254`** and owns `:53` (DNS) and `:3000` (dashboard).
- **Every other service** (Vaultwarden, Jellyfin, JDownloader2, monitoring)
  attaches to the shared **`traefik` bridge** network and is reverse-proxied by
  Traefik. They are normal bridge backends; only Traefik + AdGuard share `.254`.

## Repository layout

```tree
vesta/
├── README.md
├── LICENSE
├── .gitignore
├── .env.example            # required secrets, no values
├── docs/
│   ├── architecture.md     # macvlan + shared-netns design; why no Tailscale here
│   ├── traefik.md          # bridge, macvlan, Cloudflare DNS-01, routing, dashboards
│   ├── adguard.md          # shared netns, DNS rewrites, LAN + tailnet DNS
│   ├── services.md         # how to add a service; per-service notes
│   ├── operations.md       # bring-up order, the netns-restart gotcha, backups
│   └── disaster-recovery.md
├── traefik/
│   ├── compose.yaml        # macvlan + bridge owner, @ 192.168.50.254
│   ├── traefik.yml         # static config (entrypoints, Cloudflare resolver)
│   ├── config/
│   │   ├── tls.yml         # default wildcard cert via Cloudflare DNS-01
│   │   └── adguard.yml     # file-provider route to AdGuard dashboard (loopback)
│   └── le/                 # ACME store (acme.json) — gitignored
├── adguard/compose.yaml    # network_mode: container:traefik  ⇒ same .254
├── vaultwarden/compose.yaml
├── jellyfin/compose.yaml
├── jdownloader/compose.yaml
└── monitoring/compose.yaml # Uptime Kuma example
```

## Quick start

```sh
cp .env.example .env        # fill in secrets (NOT committed)
docker network create traefik --subnet 172.30.0.0/24   # the external bridge, once
docker compose -f traefik/compose.yaml up -d            # netns owner FIRST
docker compose -f adguard/compose.yaml up -d            # joins Traefik's netns
docker compose -f vaultwarden/compose.yaml up -d        # then the bridge backends
# ...jellyfin, jdownloader, monitoring
```

Order matters and `depends_on` does not cross compose files — see
`docs/operations.md`.

## Secrets / before you publish

`.env` is gitignored; only `.env.example` (no values) is committed. Required
secrets: Cloudflare API tokens (Traefik DNS-01) and the Vaultwarden admin token.
The Traefik ACME store (`traefik/le/acme.json`) contains private keys and is
gitignored. The only environment specifics committed are an RFC 1918 subnet and
the public service domain (already in Certificate Transparency logs).

## License

[MIT](./LICENSE).
