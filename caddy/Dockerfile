# Caddy built with the Cloudflare DNS provider.
# Needed because our internal names (*.chicago.sing.sh and the sing.sh
# overrides) don't resolve publicly, so ACME must use the DNS-01 challenge
# via the Cloudflare API rather than HTTP-01. The stock caddy image does
# not bundle DNS providers, hence this xcaddy build.
FROM caddy:2-builder AS builder
RUN xcaddy build --with github.com/caddy-dns/cloudflare

FROM caddy:2
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
