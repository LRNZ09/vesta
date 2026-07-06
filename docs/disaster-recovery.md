# Disaster recovery & rebuild

## Secrets inventory (kept OUT of git)

| Secret | Used by | Where it really lives |
| --- | --- | --- |
| `CF_DNS_API_TOKEN` / `CF_ZONE_API_TOKEN` | Traefik DNS-01 | Cloudflare dashboard; `services/traefik/.env` |
| `VAULTWARDEN_ADMIN_TOKEN` | Vaultwarden `/admin` | `services/vaultwarden/.env` / password manager |
| AdGuard admin password | AdGuard UI login | redacted in `conf/AdGuardHome.yaml`; set on deploy |
| DNSCrypt keys (if enabled) | AdGuard inbound DNSCrypt | `services/adguardhome/conf/dnscrypt.yaml` (gitignored) |
| MyJDownloader login | JDownloader2 | entered in the JD2 web UI |
| `services/traefik/le/acme.json` | issued certs + keys | runtime; optionally backed up encrypted |

Secrets are **per-service `.env` files** (each service commits only its
`.env.example`), plus the redacted/gitignored config noted above.

## ⚠ Vaultwarden chicken-and-egg

If your passwords (including the Cloudflare tokens and the Vaultwarden admin
token) live **inside** Vaultwarden, and Vaultwarden is down, you cannot bootstrap
the stack that hosts it. Keep the **rebuild-critical** secrets — Cloudflare
tokens, Vaultwarden admin token, and a recent **Vaultwarden data export** — in an
independent location (an offline copy / a second password manager), not only in
the Vaultwarden you're trying to restore.

## Rebuild from zero

1. On DSM, install **Container Manager** (Docker).
2. `git clone` this repo. Copy the **root** `.env.example → .env` (set `UID`/`GID`
   for the host user), and each service's `.env.example → .env` where present
   (Traefik and Vaultwarden). Compose commands take `--env-file .env` (see
   `operations.md`).
3. Restore the bind-mount state from backup — especially
   `services/vaultwarden/data/` and `services/adguardhome/conf/`.
4. Set a real admin password hash in `conf/AdGuardHome.yaml`. If you use inbound
   DNSCrypt, also copy `conf/dnscrypt.yaml.example → conf/dnscrypt.yaml` and fill
   in the keys.
5. Bring stacks up in order (`operations.md`): Traefik (creates the network) →
   AdGuard → bridge backends.
6. If `acme.json` wasn't restored, Traefik re-issues the wildcard via DNS-01 —
   ensure the Cloudflare tokens are valid first.
7. If AdGuard `conf/` wasn't restored, re-run first-run setup: bind the UI to
   `172.50.0.1:3080`, DNS to `:53`, and re-add the rewrite
   `*.admin.lorenzopieri.dev → 192.168.50.254`.

## What this repo does and doesn't restore

- **Does:** the topology, every compose file, Traefik static/dynamic config, the
  AdGuard config, and the operational runbook — i.e. a copy-paste rebuild of the
  *configuration*.
- **Doesn't:** the data (bind-mount dirs) or the secrets (`.env`, DNSCrypt keys,
  the admin password hash) — those are backed up separately, by design, so this
  repo can be public.
