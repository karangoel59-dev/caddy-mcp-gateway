# caddy-mcp-gateway

Caddy config that puts several MCP servers behind one HTTPS hostname.

Caddy terminates TLS and forwards `/<name>/mcp` to a single local MCP proxy
listening on `127.0.0.1:18781`. Every other path returns `404`.

## Routes

| Path | Upstream |
|---|---|
| `/ssh/mcp`, `/kgmail/mcp`, `/git/mcp`, `/ponytail/mcp`, `/github-dev/mcp`, `/github/mcp`, `/gdrive/mcp`, `/context7/mcp`, `/playwright/mcp`, `/task360/mcp` | `127.0.0.1:18781` |
| anything else | `404 Not found` |

## Usage

1. Replace `mcp.example.com` in `Caddyfile` with a hostname that resolves to your server.
2. Edit the `@mcp` path list to match the servers your proxy exposes.
3. Install and reload:

   ```sh
   sudo cp Caddyfile /etc/caddy/Caddyfile
   sudo systemctl reload caddy
   ```

Ports 80 and 443 must be reachable for certificate issuance.

## Notes

- `flush_interval -1` disables response buffering so streamed MCP responses are delivered immediately.
- `disable_tlsalpn_challenge` makes Caddy use the HTTP-01 challenge on port 80.
- Caddy adds no authentication here. The upstream proxy must authenticate requests itself.
- Tested with Caddy 2.6.2.
