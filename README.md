# wg-vpn-gateway

Sends the **whole VM's** outbound traffic (the host itself, Docker containers,
and clients of any VPN server running on the box) through a WireGuard tunnel
from your own config file, while everything local keeps working.

## Install / update (same command for both)

```bash
curl -fsSL -o /tmp/wg-vpn-gateway.zip https://raw.githubusercontent.com/AngelGonePro/wg-vpn-gateway/refs/heads/main/wg-vpn-gateway.zip && python3 -c "import zipfile;open('/tmp/wg-vpn-deploy.sh','wb').write(zipfile.ZipFile('/tmp/wg-vpn-gateway.zip').read('wg-vpn-gateway/deploy-update.sh'))" && bash /tmp/wg-vpn-deploy.sh /tmp/wg-vpn-gateway.zip ~/wg-vpn-gateway
```

1. **First run** puts the project in `~/wg-vpn-gateway` and stops, because
   there's no WireGuard config yet.
2. Put your client config at `~/wg-vpn-gateway/config/wg.conf`. Any
   provider's wg-quick file works as-is. Optionally adjust `.env`.
3. **Run the same command again.** It builds and starts the stack, then waits
   for the tunnel to pass traffic.

Every later run of the command upgrades in place. `.env` and `config/wg.conf`
are never overwritten. Any other file you've edited is left alone and
reported. New `.env` keys are appended with their defaults. If the tunnel
doesn't come up healthy, the gateway is stopped again so the VM goes back to
direct routing rather than pointing at a dead tunnel.

From inside an existing install you can also run
`./deploy-update.sh https://raw.githubusercontent.com/AngelGonePro/wg-vpn-gateway/refs/heads/main/wg-vpn-gateway.zip .`

## Kill switch (`KILLSWITCH=true`, the default)

Whenever the tunnel isn't up (gateway stopped, crashed, mid-upgrade, VPN
down), outbound traffic is **blocked** instead of using this VM's real IP. That
covers the host, Docker containers and clients of VPN servers on the box, and
the block stays in place even after `docker compose down`. Still working
while blocked:

- inbound connections (SSH, VPN servers, published ports) and their replies,
  including the SSH session you're in
- local networks: LAN, Docker containers ↔ host, VPN clients ↔ LAN/host,
  link-local and multicast
- `BYPASS_OUT_TCP_PORTS` / `BYPASS_OUT_UDP_PORTS` and `BYPASS_DESTINATIONS`

So you can't lock yourself out. To lift the block, either set
`KILLSWITCH=false` and run the install/update command, or:

```bash
cd ~/wg-vpn-gateway && docker compose run --rm wg-vpn-gateway release
```

If your config's `Endpoint` is a hostname, only that one lookup goes out
directly while the tunnel comes up. All other DNS goes through the tunnel.

With `KILLSWITCH=false`, `docker compose down` removes everything and
traffic goes direct again.

## What goes where

| Traffic | Path |
|---|---|
| Outbound from the host / containers / local VPN clients | **Tunnel** |
| Any connection arriving on the WAN (SSH, published Docker ports, SoftEther/WireGuard servers, nginx, frps…), including ones already open when the gateway starts | Real IP |
| Outbound to `BYPASS_OUT_TCP_PORTS` / `BYPASS_OUT_UDP_PORTS` (default: TCP 22) | Real IP |
| Outbound to `BYPASS_DESTINATIONS` (other VPSes, or local networks only reachable through your router) | Real IP / normal network |
| Any subnet the host routes directly (LAN, Docker networks, VPN-server client subnets) | Local, unchanged |
| Link-local (169.254.x.x, e.g. cloud metadata), multicast (mDNS/SSDP), broadcast | Local, unchanged |
| Routes owned by tools with their own earlier rules (e.g. Tailscale) | Unchanged |

So `ssh -J this-vps other-vps`, scp between boxes, and SSH into this box all
keep using real IPs, and other VPSes that allowlist this box's IP still see it.

## How it works

`Table = off` + `FwMark` are injected into a copy of your config, then:

1. `ip rule` (priorities `RULE_PRIORITY`…`+5`): fwmark → main, bypass
   destinations → main, main minus its default route, default → tunnel table,
   then (kill switch) → `lo` with rejects, then `unreachable` as a backstop.
2. iptables `mangle` (nftables backend via iptables-nft): connections seen on
   the WAN, the tunnel's own packets and bypass-port connections get a
   connmark that routes them via main.
3. iptables `nat`: forwarded traffic leaving through the tunnel is masqueraded
   to the tunnel address. A bypass-marked socket that picked the tunnel
   address is masqueraded to the WAN address.

`net.ipv4.conf.all.src_valid_mark=1` is set so hosts with strict `rp_filter` work.

## Notes

- Tailscale's own encrypted traffic (if installed) uses its own routing
  rules ahead of this gateway's, so it keeps using the real IP.
- The tunnel interface defaults to `wgvpn` so it can't collide with a
  WireGuard server interface such as `wg0`.
- The host keeps its own DNS servers. Their upstream lookups go through the tunnel.
