# Traefik

Traefik is the TLS-terminating reverse proxy and the **owner** of the macvlan
address `.254`. All configuration lives under `services/traefik/`.

## Networks

- **`home_lan`** (macvlan, parent `eth0`, `192.168.50.0/24`, gw `.1`): defined
  in `services/traefik/compose.yaml`; gives Traefik the fixed address
  `192.168.50.254`.
- **`traefik`** (bridge, `172.50.0.0/24`, gateway `172.50.0.1`): the network
  every backend attaches to. It is **defined in `services/traefik/compose.yaml`**
  (named `traefik`), so bringing Traefik up first creates it; the other services
  reference it as `external: true`. No manual `docker network create` is needed
  as long as Traefik starts before the backends.

## Static config — `services/traefik/traefik.yaml`

- Entrypoints `web` (`:80`, redirects to HTTPS) and `websecure` (`:443`).
- `websecure` applies the **`lan-only@file`** middleware globally, so every
  proxied service is reachable only from the LAN (see below).
- An ACME resolver named **`le`** using the Cloudflare **DNS-01** challenge
  (tokens from `.env`: `CF_DNS_API_TOKEN`, `CF_ZONE_API_TOKEN`;
  storage `/le/acme.json`).
- Providers: `docker` (label discovery, `exposedByDefault: false`, restricted to
  the `traefik` network) and `file` (dynamic config in `/config`).

## Certificates — `services/traefik/config/tls.yaml`

A **default generated wildcard** certificate for `*.admin.lorenzopieri.dev` via
the DNS-01 resolver `le`. Because it is the default cert, individual routers need
only `tls: "true"` and automatically get the wildcard — no per-router cert
config. The resolver named here must match the one defined in `traefik.yaml`.

## Access control — `services/traefik/config/lan-only-middleware.yaml`

The `lan-only` middleware is an `ipAllowList` (`192.168.50.0/24` + loopback) and
is attached to the whole `websecure` entrypoint in `traefik.yaml`. This — not
basic auth — is what keeps the dashboards and admin UIs private. Uncomment the
`100.64.0.0/10` range to also admit tailnet clients. (A `BASIC_AUTH_USER_HASH`
slot exists in `.env.example` for a future basic-auth middleware, but none is
wired up today.)

## Routing a bridge service (Docker labels)

```yaml
labels:
  traefik.enable: "true"
  traefik.http.routers.<name>.rule: Host(`<name>.admin.lorenzopieri.dev`)
  traefik.http.routers.<name>.entrypoints: websecure
  traefik.http.routers.<name>.tls: "true"
  traefik.http.services.<name>.loadbalancer.server.port: "<container-port>"
networks: [traefik]
```

`loadbalancer.server.port` is the port **inside** the container; Traefik reaches
it over the bridge. For an HTTPS backend with a self-signed cert (e.g.
JDownloader2), add `loadbalancer.server.scheme: https` and
`loadbalancer.serverstransport: insecure-https-transport@file` — the transport
that skips verification is defined in
`config/insecure-https-transport.yaml`.

## Dashboards

- **Traefik dashboard:** enabled via `api.dashboard: true` and routed by the
  file provider (`config/dashboard.yaml`, `service: api@internal`) at
  `traefik.admin.lorenzopieri.dev`. It is protected by the global `lan-only`
  middleware, not published open.
- **AdGuard dashboard:** AdGuard runs in host mode and binds its UI to the
  `traefik` bridge gateway `172.50.0.1:3080`, so it is reachable from Traefik but
  not from the LAN. A **file provider** route (`config/adguardhome.yaml`) points
  at `http://172.50.0.1:3080` and serves it at
  `adguardhome.admin.lorenzopieri.dev`.

## Image tag

`services/traefik/compose.yaml` pins `traefik:langres` (a locally mirrored tag).
If you don't have that mirror, use an upstream versioned tag such as `traefik:v3`.
