# SearXNG + mcp-searxng behind an existing host nginx

Path-based layout, designed to ride on an **existing** nginx vhost with an
**existing** certificate (no extra subdomains or certs needed):

```
               ┌─ https://<domain>/searxng ─► 127.0.0.1:SEARXNG_PORT ─► searxng:8080
host nginx ────┤
(TLS, certs)   └─ https://<domain>/mcp    ─► 127.0.0.1:MCP_PORT     ─► mcp-searxng:3000
                                                             │
                                                            └► 127.0.0.1:8080 (internal)

searxng, mcp-searxng (gluetun network stack) ─► ProtonVPN ─► search engines, web_url_read targets
```

| Service       | Purpose                                                     |
|---------------|-------------------------------------------------------------|
| `searxng`     | Metasearch engine (web UI + JSON API), limiter enabled      |
| `valkey`      | Storage for SearXNG's limiter / bot detection               |
| `mcp-searxng` | MCP server, Streamable HTTP, hardened mode (bearer token)   |
| `gluetun`     | ProtonVPN WireGuard tunnel, sole egress of searxng + mcp    |

## Files

```
docker-compose.yml
.env.example              → copy to .env
searxng/settings.yml      SearXNG overlay (json format on, limiter on)
searxng/limiter.toml      trusts the Docker gateway; passes the MCP container
nginx/searxng-mcp.conf    locations to include into an existing HTTPS vhost
DEPLOYMENT.md             (untracked) machine-specific deploy instructions
```

## Setup

1. Configure and start:
   ```sh
   cp .env.example .env
   sed -i "s/^SEARXNG_SECRET=.*/SEARXNG_SECRET=$(openssl rand -hex 32)/" .env
   sed -i "s/^MCP_AUTH_TOKEN=.*/MCP_AUTH_TOKEN=$(openssl rand -hex 32)/" .env
   $EDITOR .env          # MCP_DOMAIN, SEARXNG_BASE_URL, ports, PROTON_WG_PRIVATE_KEY, VPN_COUNTRIES
   docker compose up -d
   ```
   `PROTON_WG_PRIVATE_KEY` is the `PrivateKey` of a WireGuard config generated at
   account.proton.me → VPN → WireGuard (any server; the key works for all of them).
   Prerequisite for gluetun: `/dev/net/tun` must exist on the host.
2. Host nginx: include the locations in the HTTPS server block of your domain
   (certificate must already exist for that vhost):
   ```sh
   sudo mkdir -p /etc/nginx/searxng-mcp
   sudo cp nginx/searxng-mcp.conf /etc/nginx/searxng-mcp/locations.conf
   # in your domain's HTTPS server block, before "location /", add:
   #     include /etc/nginx/searxng-mcp/locations.conf;
   sudo nginx -t && sudo systemctl reload nginx
   ```

## Ports (`.env`)

| Variable       | Default     | Meaning                                              |
|----------------|-------------|------------------------------------------------------|
| `BIND_ADDRESS` | `127.0.0.1` | Host interface the two ports are published on        |
| `SEARXNG_PORT` | `8080`      | SearXNG → `proxy_pass http://127.0.0.1:<port>`       |
| `MCP_PORT`     | `3000`      | mcp-searxng → `proxy_pass http://127.0.0.1:<port>`   |

Container-side ports stay fixed (8080 / 3000); only the host side changes.
Keep `BIND_ADDRESS=127.0.0.1` so nothing bypasses nginx. Docker's published ports
skip `ufw`/firewalld rules, so `0.0.0.0` would really be public.

## Connecting an MCP client

Endpoint `https://<MCP_DOMAIN>/mcp`, header `Authorization: Bearer <MCP_AUTH_TOKEN>`.

```sh
claude mcp add --transport http searxng https://example.com/mcp \
  --header "Authorization: Bearer $MCP_AUTH_TOKEN"
```

```json
{
  "mcpServers": {
    "searxng": {
      "type": "streamable-http",
      "url": "https://example.com/mcp",
      "headers": { "Authorization": "Bearer <MCP_AUTH_TOKEN>" }
    }
  }
}
```

opencode (`~/.config/opencode/opencode.json`):

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "searxng": {
      "type": "remote",
      "url": "https://example.com/mcp",
      "oauth": false,
      "headers": { "Authorization": "Bearer <MCP_AUTH_TOKEN>" }
    }
  },
  "tools": { "websearch": false }
}
```

## Verify

```sh
 curl -s "http://127.0.0.1:${SEARXNG_PORT}/search?q=test&format=json" | head -c 300 # JSON API, direct
curl -s https://example.com/searxng/config | head -c 200                          # via nginx
curl -s http://127.0.0.1:${MCP_PORT}/health                                        # local only (not routed via nginx)
curl -s -o /dev/null -w '%{http_code}\n' -X POST https://example.com/mcp           # expect 401
docker logs searxng-gluetun 2>&1 | grep 'Public IP address'                        # VPN exit, not the server
```

## Notes

- **Path layout.** SearXNG is served at `https://<domain>/searxng`, the MCP endpoint
  at `https://<domain>/mcp`. The `/searxng` prefix is stripped inside SearXNG itself
  (`X-Script-Name` header + `SEARXNG_BASE_URL`, see flaskfix.py), so the path must
  stay identical in `nginx/searxng-mcp.conf`, `.env` and `MCP_DOMAIN`'s vhost.
- **Proxy trust.** Requests from host nginx arrive at the containers from the Docker
  gateway `172.30.0.1` (fixed by the subnet in `docker-compose.yml`). Only that address
  is trusted for `X-Forwarded-For` (`limiter.toml`, `MCP_HTTP_TRUST_PROXY`). nginx
  overwrites the header with `$remote_addr`, so clients can't spoof their IP. If you
  change the subnet, update both places.
- **Limiter vs. MCP.** mcp-searxng calls SearXNG via 127.0.0.1 without browser headers
  and would be flagged as a bot, so `127.0.0.1` is on SearXNG's `pass_ip`.
  The MCP endpoint has its own rate limits (`MCP_RATE_*`) and bearer token.
- **VPN egress.** searxng and mcp-searxng use gluetun's network stack, so all their
  outgoing traffic, DNS included, goes through ProtonVPN; gluetun's firewall blocks it
  while the tunnel is down. Engines see a Proton exit IP, not the server's. Google and
  Bing challenge VPN exits more often; SearXNG then suspends them for a while.
- **Recreating gluetun.** If the gluetun container is restarted or recreated, searxng
  and mcp-searxng lose all networking until they are recreated too:
  `docker compose up -d --force-recreate searxng mcp-searxng`.
- **Host header.** nginx sends `Host: $host` (no port), which must equal `MCP_DOMAIN`,
  or mcp-searxng returns 403 in hardened mode.
- Pin `SEARXNG_VERSION` / `MCP_SEARXNG_VERSION` for reproducible deploys.

References: [mcp-searxng self-hosted guide](https://github.com/ihor-sokoliuk/mcp-searxng/blob/main/docs/self-hosted-searxng.md),
[mcp-searxng CONFIGURATION.md](https://github.com/ihor-sokoliuk/mcp-searxng/blob/main/CONFIGURATION.md),
[SearXNG container install](https://docs.searxng.org/admin/installation-docker.html),
[SearXNG limiter](https://docs.searxng.org/admin/searx.limiter.html),
[gluetun ProtonVPN](https://github.com/qdm12/gluetun-wiki/blob/main/setup/providers/protonvpn.md).
