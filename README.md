# wg-vpn-gateway

Sends a VM's outgoing traffic, or a whole group of VMs' traffic, through
one WireGuard VPN. Local networks, inbound connections and chosen bypasses
keep working. On top of that it adds:

- a kill switch
- threat blocking (blocklists plus Suricata)
- alerts to Discord and Uptime Kuma
- small, size-capped logs
- verified self-updates with rollback

## Install / update (same command for both)

```bash
curl -fsSL -o /tmp/wg-vpn-gateway.zip https://github.com/AngelGonePro/wireguard-upstream-vpn/raw/refs/heads/main/wg-vpn-gateway.zip && python3 -c "import zipfile;open('/tmp/wg-vpn-deploy.sh','wb').write(zipfile.ZipFile('/tmp/wg-vpn-gateway.zip').read('wg-vpn-gateway/deploy-update.sh'))" && bash /tmp/wg-vpn-deploy.sh /tmp/wg-vpn-gateway.zip ~/wg-vpn-gateway
```

The first run puts the project in `~/wg-vpn-gateway` and stops. Set it up
(below), then run the same command again to start it. Every later run
upgrades in place:

- `.env`, `config/wg.conf`, `state/` and `logs/` are never overwritten.
- Files you edited are left alone and reported.
- New `.env` keys are added with their defaults.
- If the tunnel doesn't come up healthy, the gateway is stopped. With the kill
  switch on, it stays blocked rather than using your real IP.

## Modes (`MODE` in `.env`)

| Mode | Use it on | Needs |
|---|---|---|
| `standalone` (default) | a single VM/VPS that tunnels only itself (plus its Docker containers and VPN-server clients such as StratumVPN) | `config/wg.conf` |
| `gateway` | the one VM that holds the WireGuard config for a group of VMs | `config/wg.conf`, `GATEWAY_CLIENTS` |
| `client` | every other VM on the same local network | `GATEWAY_IP` |

### One VPN for several VMs

1. **Gateway VM:** set `MODE=gateway`, put the WireGuard config at
   `config/wg.conf`, and list the VMs allowed to use it, for example
   `GATEWAY_CLIENTS=10.0.0.50=jellyfin,10.0.0.152=minecraft`. Any other device
   that tries to route through the gateway is refused.
2. **Each client VM:** set `MODE=client`, `GATEWAY_IP=<gateway VM's LAN IP>`
   and `COMPOSE_PROFILES=autoupdate` (Suricata runs on the gateway, not on
   clients).
3. Run the install command on each VM.

Client VMs keep their LAN, their inbound port forwards and their bypasses.
They have their own kill switch: if the gateway VM is off or its VPN is
down, the client is blocked, and it never falls back to your router.

A device that can't run Docker can also just set its default gateway to the
gateway VM's IP, as long as that IP is in `GATEWAY_CLIENTS`. It's protected
by the gateway's kill switch, but it gets no bypasses of its own.

## What goes where

| Traffic | Path |
|---|---|
| Outgoing from the VM, its containers, and its VPN-server clients | **VPN** (client VMs: via the gateway VM) |
| Any connection arriving from outside, including ones already open when the gateway starts | Real IP |
| `BYPASS_OUT_TCP_PORTS` / `BYPASS_OUT_UDP_PORTS` (default: TCP 22) | Real IP |
| `BYPASS_DESTINATIONS` (other VPSes, or local networks only reachable through your router) | Real IP / normal network |
| `SPLIT_TUNNEL_DESTINATIONS` | The second WireGuard config |
| Directly connected networks (LAN, Docker networks, VPN-server client subnets), link-local, multicast, broadcast | Local, unchanged |

## Split tunnel: a second WireGuard config for chosen IPs

Everything goes through the main config (`config/wg.conf`, e.g. a fast VPN
provider), except the IPs in `SPLIT_TUNNEL_DESTINATIONS`, which go through a
second config (`config/wg-split.conf`, e.g. your own server's WireGuard):

```
SPLIT_TUNNEL_DESTINATIONS=192.168.209.180,45.79.25.46
```

- In gateway mode the client VMs use it too; nothing to set on them.
- If the second tunnel is down, those IPs can't be reached until it's back.
  They're never sent from your real IP.
- Bypasses still win. With the default `BYPASS_OUT_TCP_PORTS=22`, SSH to
  those IPs uses your real IP.
- Change the IPs in `.env` and run the install command again.

## Kill switch (`KILLSWITCH=true`)

Whenever the tunnel isn't up (gateway stopped, crashed, mid-update, VPN or
gateway VM down), outgoing traffic is **blocked** instead of using the real
IP. The block stays in place after `docker compose down`. Inbound
connections (including your SSH session), local networks and the bypasses
keep working, so you can't lock yourself out. To lift the block:

```bash
cd ~/wg-vpn-gateway && docker compose run --rm wg-vpn-gateway release
```

The install command downloads through the VPN like everything else. If the
VPN itself is broken, the download is blocked too, so run the `release`
command above first, then the install command. The kill switch goes back on
when the gateway starts.

The gateway counts as healthy only when the VPN server answers *and* traffic
comes back through it (`HEALTH_PING_TARGET`). A VPN server that's up but not
forwarding anything shows as down in the install output, Discord and Kuma.

## Threat blocking (standalone / gateway)

UniFi can't inspect traffic once it's inside the VPN, so the gateway checks
it before encryption.

- **What blocks, and when (`BLOCK_MODE=auto`):**
  - **Right away:** trusted sources built to have no false positives —
    Spamhaus DROP, ThreatFox at confidence ≥ 90, and Suricata rules rated
    both `confidence High` and `signature_severity Major/Critical`.
  - **Never:** anything lower-confidence (other Suricata rules, lower
    ThreatFox entries). These only alert.
  - **Monitor first:** your own `EXTRA_BLOCKLIST_URLS` lists, since their
    quality is unknown. They're logged and alerted for `MONITOR_DAYS`
    (default 7), then they block, and you get a summary of what they matched.
  - `BLOCK_MODE=monitor` makes everything alert-only. `BLOCK_MODE=block`
    makes your extra lists block right away too. Add anything legitimate to
    `ALLOWLIST`.
- **Blocklists.** These are checked on every new connection, in both
  directions:
  - [Spamhaus DROP](https://www.spamhaus.org/blocklists/do-not-route-or-peer/):
    networks controlled by criminals, built to contain no legitimate traffic.
  - Optional [abuse.ch ThreatFox](https://threatfox.abuse.ch/) malware
    command-and-control `ip:port` entries (exact port only, confidence ≥ 90).
    Needs a free key in `THREATFOX_AUTH_KEY`.
  - Optional lists of your own in `EXTRA_BLOCKLIST_URLS`.
- **Safety checks.**
  - These are never blocked, whatever a list says: private/local ranges,
    `ALLOWLIST`, bypass destinations, your DNS servers, the VPN endpoint,
    the gateway VM and its clients.
  - These list entries are ignored: entries wider than /8 (IPv4) or /16
    (IPv6), and entries that would touch any of the protected addresses above.
  - A list download that comes back empty, shrinks by more than half or
    fails is rejected. The previous copy stays in force and you get an alert.
- **Suricata** (`COMPOSE_PROFILES` includes `suricata`) inspects traffic
  entering and leaving the tunnel, using Emerging Threats Open rules in
  `SURICATA_CATEGORIES`.
  - **What blocks:** a rule blocks (right away) only when ET rates it *both*
    `confidence High` ("minimal likelihood of false positives") and
    `signature_severity Major/Critical`. Everything else only alerts.
  - **False positives:** put a rule's sid in `SURICATA_ALERT_ONLY_SIDS` (stop
    it blocking) or `SURICATA_DISABLE_SIDS` (drop it).
  - **Fails open:** if Suricata is down or out of memory, traffic passes.
    The kill switch is separate.
  - **Full speed (`SURICATA_BYPASS=true`):** Suricata inspects the part of
    each connection its rules can match, then the rest of the connection
    skips it. That part is the requests, hostnames, the TLS handshake and
    certificate, and the first `SURICATA_STREAM_DEPTH` of plain traffic.
    After a TLS handshake everything is encrypted and no rule can match
    anyway. On a 300 MB download, Suricata handled about 700 packets instead
    of over 500,000.
- NextDNS (or whatever DNS filtering your VMs use) keeps working through the
  VPN. Inbound and bypassed traffic still go through UniFi as before.

Check status: `docker compose exec wg-vpn-gateway python3 /usr/local/lib/wg-vpn-gateway/guardlib.py status`

## Alerts and logs

- **Discord** (`DISCORD_WEBHOOK_URL`): blocks and would-be blocks, Suricata
  alerts, tunnel down/up, list problems, and update results. Messages are
  queued and delivered later if the VM is offline or blocked. To ping a
  role with every alert, put its ID in `DISCORD_PING_ROLE_IDS`. Only those
  roles are pinged, never @everyone.
- **Uptime Kuma** (`KUMA_PUSH_URL`, a Push monitor): a heartbeat every
  minute. It reports down when the tunnel isn't working, and Kuma also
  alerts when heartbeats stop arriving.
- **Logs** in `./logs`: each file is capped at `LOG_MAX_KB` with `LOG_KEEP`
  old copies, so the folder stays at a few MB at most. Docker's own logs are
  capped at 2 × 1 MB per container.

## Updates (`COMPOSE_PROFILES` includes `autoupdate`)

| What | When | How |
|---|---|---|
| Blocklists | daily | sanity-checked, previous copy kept on failure |
| Suricata rules | daily | tested before going live. Rules this Suricata version can't parse are skipped and reported. If the test still fails, the previous rules stay. |
| Image (Alpine, Suricata, WireGuard tools, all packages) | weekly (`UPDATE_DAY`, `UPDATE_HOUR`) | rebuilt, health-checked, rolled back and alerted on failure |
| Alpine release | with the weekly update | moves to each new stable release after verification (`ALPINE_AUTO_UPGRADE`). Older releases stop getting security fixes for the packages used here. |

The updater needs the Docker socket to rebuild the other containers. Remove
`autoupdate` from `COMPOSE_PROFILES` if you'd rather update by hand. The
VM's own OS isn't covered; the install command tells you if Debian's
`unattended-upgrades` isn't enabled.

## Memory (measured)

| Container | Typical | Cap (`.env`) |
|---|---|---|
| gateway (routing, kill switch, guard) | ~10 MB, ~35 MB briefly once a minute | `MEM_LIMIT=128m` |
| suricata | ~95 MB with a 12,000-rule set | `SURICATA_MEM_LIMIT=256m` |
| updater | a few MB idle, more while building | `UPDATER_MEM_LIMIT=256m` |

## Notes

- `gateway`/`client` mode needs the VMs on the same local network as each
  other. A remote VPS still needs its own `standalone` install.
- With no `MTU =` line in your WireGuard config, the tunnel MTU is set to
  the MTU of the interface that reaches the VPN server minus 80 (wg-quick's
  usual rule). Add `MTU = 1380` (or similar) under `[Interface]` to choose
  your own.
- The tunnel interface defaults to `wgvpn`, so it can't collide with a
  WireGuard server interface such as `wg0`.
- Tailscale's own encrypted traffic, if installed, uses its own routing rules
  ahead of this gateway's, so it keeps using the real IP.
- Blocklist data © The Spamhaus Project, and abuse.ch where enabled.
