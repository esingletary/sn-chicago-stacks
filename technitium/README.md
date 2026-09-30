# Technitium DNS — SingNet Chicago

Split-horizon DNS for the home network. Runs as a Docker stack on the Pi.
All config/data lives under this folder (`./data`) so the whole tree moves to
NVMe later as a single `rsync`.

## Stack

- `compose.yaml` — service definition. Image `technitium/dns-server:latest`.
- `./data` → `/etc/dns` in the container (all server config + zone files).

Manage:
```
cd /opt/stacks/technitium
docker compose up -d        # start
docker compose logs -f      # follow logs
docker compose pull && docker compose up -d   # update image
```

Ports published on the host:
- `53/udp`, `53/tcp` — DNS
- `5380/tcp` — web console

DoT (853) / DoH (443) are intentionally off for now.

## Host facts (discovered 2026-06-19)

| Key | Value |
|---|---|
| Owner | `es` |
| Pi LAN IP | `10.0.1.6` |
| Pi Tailscale IP | `100.90.110.37` |
| Gateway | `10.0.1.1` |
| LAN subnet | `10.0.1.0/24` |

Port 53 was free on the host (systemd-resolved inactive; only avahi on 5353).
`/etc/resolv.conf` is Tailscale-managed and is deliberately **left untouched**.

## ⚠️ Remote-operator safety

The operator is remote over Tailscale. Do NOT make Technitium the network's
sole/default DNS and do NOT change the host resolver. Validate by querying the
container directly. The UniFi DHCP + Tailscale split-DNS cutover is **manual and
deferred** until validation passes.

## Web-console configuration (one-time)

Console: http://10.0.1.6:5380  (also http://100.90.110.37:5380 over Tailscale)
First login: `admin` / `admin` — **change the password immediately** after.

### 1. Forwarders → Cloudflare DoH
Settings → **Forwarders** tab:
- Forwarder Protocol: **Https**
- Use "Quick Select Forwarders" → **Cloudflare (DNS-over-HTTPS)**, which fills:
  - `https://cloudflare-dns.com/dns-query (1.1.1.1)`
  - `https://cloudflare-dns.com/dns-query (1.0.0.1)`
- Save Settings.

### 2. Primary zone `chicago.sing.sh` (internal-only)
Zones → Add Zone → name `chicago.sing.sh`, type **Primary Zone**. Add A records:

| Name | Type | Value |
|---|---|---|
| `gateway` | A | `10.0.1.1` |
| `switch`  | A | `10.0.1.2` |
| `ap`      | A | `10.0.1.3` |
| `dns`     | A | `10.0.1.6` |
| `*`       | A | `10.0.1.6` (Caddy / Pi) |

Explicit records win over the `*` wildcard, so the named hosts above resolve to
their own IPs and everything else under the zone falls to Caddy.

### 3. `sing.sh` selective overrides (keep apex public)
Do NOT make Technitium authoritative for all of `sing.sh`.

a. Zones → Add Zone → name `sing.sh`, type **Conditional Forwarder Zone**
   - Protocol **Https**, forwarder `https://cloudflare-dns.com/dns-query (1.1.1.1)`
   - This forwards everything under `sing.sh` to Cloudflare (public records intact).

b. Zones → Add Zone → name `zs.sing.sh`, type **Primary Zone**
   - Add A record, name `@` → `10.0.1.6` (Caddy / Pi)
   - More specific than the `sing.sh` forwarder, so it wins. Repeat this pattern
     (one per-FQDN primary zone) for any future internal override.

## Validation (run from the host, no cutover)

```
dig @10.0.1.6 gateway.chicago.sing.sh   # → 10.0.1.1
dig @10.0.1.6 zs.sing.sh                # → 10.0.1.6
dig @10.0.1.6 sing.sh                   # → public IP (forwarded to Cloudflare)
dig @10.0.1.6 example.com               # → normal public resolution
```

## Out of scope — manual, operator, AFTER validation

- UniFi: DHCP DNS → `10.0.1.6`; add a DHCP reservation for the Pi.
- Tailscale admin → DNS: add `100.90.110.37` as a **split-DNS** nameserver
  restricted to `chicago.sing.sh` **and** `sing.sh` (both must route to the Pi
  so the `sing.sh` overrides apply remotely).
- Confirm the `10.0.1.0/24` subnet route is approved and IP forwarding is on.

## Later / optional
- Front the web console via Caddy at `dns.chicago.sing.sh → 10.0.1.6:5380`.
- `network_mode: host` for accurate client-source-IP logging (port-53 conflict
  caveat applies more strongly).
