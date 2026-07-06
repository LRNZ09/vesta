# Architecture & rationale

## The port conflict that drives the design

The NAS is a Synology running DSM, which occupies host ports **80 and 443** for
its own UI and reverse-proxy subsystem. Traefik needs those same ports. Freeing
them on DSM means editing Synology's nginx templates plus a boot script, and a
DSM update reverts it — fragile.

**Solution: give Traefik its own L2 identity.** A **macvlan** interface attaches
Traefik directly to the LAN as `192.168.50.254`, a separate address from the
NAS. Traefik then owns `:80/:443` on `.254` while DSM keeps them on the host —
no conflict, no host modification.

## AdGuard runs in host mode, independent of Traefik

AdGuard Home needs `:53`. It runs with **`network_mode: host`** and binds DNS on
`0.0.0.0:53`, so it serves the LAN on the **NAS's own host address** — not on
Traefik's `.254`. DSM does not run its own resolver on `:53`, so there is no
conflict.

The admin web UI is deliberately **not** exposed on the LAN. AdGuard binds it to
`172.50.0.1:3080` — the gateway address of the `traefik` bridge network. The
only thing that can reach that address is a container on the `traefik` bridge,
i.e. Traefik itself, which reverse-proxies the UI at
`adguardhome.admin.lorenzopieri.dev` (file-provider route, see `traefik.md`).

Running AdGuard in host mode rather than sharing Traefik's network namespace
buys one important property: **DNS is decoupled from Traefik's lifecycle.**
Recreating Traefik (image update, `down`/`up`, config change) does not touch
AdGuard's networking, so a Traefik restart never takes DNS down for the house.

## Everything else is a plain bridge backend

Vaultwarden, JDownloader2, and whoami attach to the external **`traefik`
bridge** network (`172.50.0.0/24`) and are reverse-proxied by Traefik. They do
**not** need `.254`; Traefik reaches them over the bridge by their Traefik
labels.

## Why there is no Tailscale on the NAS

(Full story in the `janus` repo.) Remote access uses Tailscale on the **router**
as a subnet router + exit node. Tailscale cannot run cleanly on this NAS because:

- macvlan has a kernel rule: a macvlan child and its **parent host cannot reach
  each other**. So host-mode Tailscale on the NAS can't reach Traefik at `.254`,
  and a co-located Tailscale (sharing a netns to get `.254`) can't reach the DSM
  host.
- With a **single NIC**, the usual escapes (second NIC, host-side shim, freeing
  DSM's ports) are unavailable or fragile.

The router is a **separate L2 device** at the gateway, so it is not the macvlan
parent and reaches `.254`, DSM, and the whole LAN without isolation. Hence
Tailscale lives there, and this NAS stays a clean Docker host.

## Topology

```text
   LAN clients ──┐                         ┌── tailnet clients
                 │                         │   (via janus subnet route)
                 ▼                         ▼
   ┌──────────────────────────────────────────────────────────────┐
   │ Synology DSM host  (single NIC: eth0)                          │
   │  DSM UI keeps host :80/:443                                    │
   │                                                                │
   │  AdGuard (network_mode: host)  :53 on 0.0.0.0  → DNS for LAN   │
   │                                                                │
   │  ┌── macvlan on eth0 ──┐   ┌── traefik bridge (172.50.0.0/24) ─┐│
   │  │ Traefik @ .254      │───│  gw 172.50.0.1                    ││
   │  │ owns :80 :443       │   │   ├─ AdGuard UI @ 172.50.0.1:3080 ││
   │  └─────────────────────┘   │   ├─ Vaultwarden                  ││
   │                            │   ├─ JDownloader2                 ││
   │                            │   └─ whoami                       ││
   │                            └───────────────────────────────────┘│
   └──────────────────────────────────────────────────────────────┘
```

Traefik lives on both the macvlan (`.254`, for `:80/:443` on the LAN) and the
`traefik` bridge (to reach backends and AdGuard's UI on the bridge gateway).
AdGuard answers DNS on the host address and **rewrites** every
`*.admin.lorenzopieri.dev` name to `.254`, so clients hit Traefik for HTTPS.
LAN clients reach `.254` over the switch; tailnet clients reach it via the
router's subnet route — same address either way, no split-horizon.
