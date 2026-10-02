# Caddy — SingNet Chicago reverse proxy

Reverse proxy + automatic HTTPS for internal services. Custom Caddy build that
includes the **Cloudflare DNS** provider so it can issue real Let's Encrypt
certs via the **ACME DNS-01** challenge — required because our names
(`*.chicago.sing.sh` and the `sing.sh` overrides) resolve **internally only**
and therefore can't use HTTP-01.

All config/data lives under this folder so the whole tree moves to NVMe later
as a single `rsync`.

## Files

- `Dockerfile` — `xcaddy build --with github.com/caddy-dns/cloudflare` (stock
  image has no DNS providers). Produces image `caddy-cloudflare:local`.
- `compose.yaml` — service def. Ports 80, 443, 443/udp (HTTP/3).
- `Caddyfile` — sites + TLS config.
- `.env` — holds `CF_API_TOKEN` (NOT committed). `.env.example` documents it.
- `./data` → `/data` — ACME account + issued certs.
- `./config` → `/config` — Caddy autosaved config.
- `./site` → `/srv` — static content.

## Manage

```
cd /opt/stacks/caddy
docker compose build        # rebuild the custom image (after Dockerfile change)
docker compose up -d
docker compose logs -f
docker compose up -d --force-recreate   # apply Caddyfile edits (see gotcha below)
```

> **Gotcha — editing the Caddyfile:** it's bind-mounted as a *single file*.
> Editors that save via atomic rename give the file a new inode, which the
> running container does **not** see — so `caddy reload` reports
> `config is unchanged` and keeps the old config. After any Caddyfile edit,
> run `docker compose up -d --force-recreate` to re-bind the mount.

## Shared `proxy` network

Caddy lives on an external Docker network named `proxy`. Other stacks join it so
Caddy can reach them by service/container name. Created once with:
```
docker network create proxy
```
A backend stack joins by adding to its compose:
```yaml
services:
  myapp:
    networks: [proxy]
networks:
  proxy:
    external: true
```
Then route to it in the Caddyfile, e.g. `reverse_proxy myapp:8080`.

## TLS: two phases

The Caddyfile defines two snippets; switch a site between them with one `import`.

- **`tls_internal`** (PHASE 1, current): Caddy's internal CA. Untrusted by
  clients (`curl -k`), but proves the proxy + HTTPS termination work without
  needing any secret.
- **`tls_cloudflare`** (PHASE 2): real Let's Encrypt wildcard certs via
  Cloudflare DNS-01. Publicly trusted, auto-renewing, works for internal names.

### Going to PHASE 2 (real certs)

1. Create a scoped Cloudflare API token (see `.env.example`): `Zone:DNS:Edit` +
   `Zone:Zone:Read`, restricted to the `sing.sh` zone. Put it in `.env`.
2. (Recommended) Uncomment `acme_ca …staging…` in the Caddyfile global block to
   validate against Let's Encrypt **staging** first (avoids burning the prod
   rate limit). Reload, confirm issuance in logs, then re-comment it.
3. Change `import tls_internal` → `import tls_cloudflare` on each site.
4. `docker compose up -d` (env change) then reload. Watch logs for
   `certificate obtained successfully … issuer: ...lets...`.

### Certificate Transparency note

Publicly-trusted certs (incl. the `*.chicago.sing.sh` wildcard) are logged in
public CT logs, so those hostnames become publicly visible even though they
never resolve publicly. Acceptable for a homelab; if you'd rather not leak
internal names, keep `tls_internal` (and distribute Caddy's root CA) instead.

## Status (2026-06-19)

- **PHASE 2 live**: real Let's Encrypt certs via Cloudflare DNS-01 (user token
  `cfut_…` with `Zone:DNS:Edit` on `sing.sh`). Validated against staging first,
  then production. Both sites use `import tls_cloudflare`; certs auto-renew.
  - `https://*.chicago.sing.sh` → 200, trusted wildcard cert (SAN
    `*.chicago.sing.sh`), HTTP→HTTPS redirect + HTTP/3.
  - `https://zs.sing.sh` → zscraper gallery (`zscraper:3001`, `/opt/stacks/zscraper`);
    `/images/*` served straight from its data dir.
  - `https://cb.sing.sh` → CB Checker (`cbchecker:3001`, `/opt/stacks/cbchecker`).
    Needs the `cb.sing.sh` override zone in Technitium (same as `zs.sing.sh`).
  - `https://dns.chicago.sing.sh` → Technitium web console (HTTP `technitium:5380`
    upstream over the `proxy` network). **Live.** Raw DNS on :53 is untouched.
  - `https://gateway.chicago.sing.sh` → UniFi UCG-Ultra UI (`https://10.0.1.1`,
    self-signed upstream via `tls_insecure_skip_verify`). Cert issued + proxy
    verified, but **pending one manual step**: remove the explicit `gateway` A
    record (→10.0.1.1) in Technitium so the wildcard (→10.0.1.6=Caddy) serves it.
    `https://10.0.1.1` remains a direct fallback.
- `restart: unless-stopped`; `docker`/`containerd` enabled on boot.
- Add more internal services as `reverse_proxy <name>:<port>` once they join the
  `proxy` network.
