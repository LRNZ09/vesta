# AdGuard Home

AdGuard is both the LAN's DNS resolver and the tailnet's, and it provides the
hostname→IP rewrites that make services reachable by name.

## Host-mode networking

`services/adguardhome/compose.yaml` uses `network_mode: host` (running as
`${UID}:${GID}`), so AdGuard has no `networks:` block. It binds:

- **DNS** on `0.0.0.0:53` — served on the **NAS's own host address**. DSM does
  not occupy `:53`, so there is no conflict.
- **Admin UI** on `172.50.0.1:3080` — the gateway of the `traefik` bridge. Only
  a container on that bridge (Traefik) can reach it, so the UI is never exposed
  directly on the LAN; it is proxied at `adguardhome.admin.lorenzopieri.dev`
  (see `traefik.md`).

Because AdGuard is host-mode and independent of Traefik's namespace, a Traefik
recreate does **not** disturb DNS — see `architecture.md`.

## Upstreams & privacy

- **Upstream:** Quad9 over DoH (`https://dns.quad9.net/dns-query`), with
  DNSSEC enabled; bootstrap `9.9.9.9`, fallback Cloudflare (`1.1.1.1`).
- **Inbound DNSCrypt** is **disabled by default** in the committed config
  (`port_dnscrypt: 0`). To enable it, provide `conf/dnscrypt.yaml` — live private
  key material, so it is **gitignored**; a redacted `conf/dnscrypt.yaml.example`
  is committed as a scaffold to copy and fill in.
- The full config is committed as a redacted scaffold,
  `conf/AdGuardHome.yaml.example`; the **live `conf/AdGuardHome.yaml` is
  gitignored** — AdGuard rewrites it at runtime and it holds the admin bcrypt
  hash. Copy the scaffold and set a real hash before deploying
  (`htpasswd -B -n -b <user> <password>`), with the container stopped.

## DNS rewrites

Two rewrites cover every service:

```text
admin.lorenzopieri.dev    ->  192.168.50.254
*.admin.lorenzopieri.dev  ->  192.168.50.254
```

Both point at **Traefik's `.254`**, so any `*.admin.lorenzopieri.dev` name
resolves to the reverse proxy for HTTPS. This is correct for **both** audiences
with no split-horizon, because both reach the same `.254`:

- **LAN clients** reach `.254` directly over the switch.
- **Tailnet clients** reach `.254` via the router's advertised subnet route.

For physical hosts, add rewrites to their real `192.168.50.x` addresses.

## Who points at AdGuard

DNS is served on the **NAS's host address** (host-mode `:53`), so that is the
resolver clients use — not `.254`.

- **LAN clients:** the router hands out the NAS's address as the DNS server via
  DHCP (configured in `janus`).
- **Tailnet clients:** the Tailscale admin console sets the global nameserver to
  the NAS's address (configured in `janus`). The router itself uses
  `--accept-dns=false` so Tailscale does not touch its dnsmasq.

## First-run setup

On a fresh deploy, seed the live config from the committed scaffold — with the
container stopped, since AdGuard overwrites the file while running:

1. `cp conf/AdGuardHome.yaml.example conf/AdGuardHome.yaml`
2. Set a real admin hash in `users[].password`
   (`htpasswd -B -n -b <user> <password>` — keep the part after the colon).
3. Copy `conf/dnscrypt.yaml.example → conf/dnscrypt.yaml` if you use DNSCrypt.

The scaffold already binds the **admin UI** to `172.50.0.1:3080` (Traefik-only),
keeps **DNS on `:53`**, and carries the `*.admin.lorenzopieri.dev → .254`
rewrites — so no setup wizard is needed. The dashboard is then reached at
`https://adguardhome.admin.lorenzopieri.dev` via the Traefik file route
(`services/traefik/config/adguardhome.yaml`).
