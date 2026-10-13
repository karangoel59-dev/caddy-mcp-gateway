# caddy-mcp-gateway

Puts several stdio MCP servers behind one HTTPS hostname.

```
client ──HTTPS──▶ Caddy :443 ──▶ supergateway 127.0.0.1:18781 ──stdio──▶ MCP servers
```

- **Caddy** terminates TLS and forwards `/<name>/mcp` to the gateway. Every other path returns `404`.
- **[supergateway](https://github.com/supercorp-ai/supergateway)** exposes each stdio MCP server as streamable HTTP and requires an API key.
- **`forward-flow-mcp-bridge`** is a Python script that generates the gateway config and controls the service.

## Files

| Path | Purpose |
|---|---|
| `Caddyfile` | HTTPS site and reverse proxy rules |
| `bin/forward-flow-mcp-bridge` | Generates the gateway config; starts and stops the service |
| `systemd/forward-flow-mcp-bridge.service` | systemd user unit that runs supergateway |
| `runtime/package.json`, `runtime/package-lock.json` | Pins supergateway 4.2.0-rc.1 |

## Setup

Replace `mcp.example.com` with a hostname that resolves to your server, in both
`Caddyfile` and `bin/forward-flow-mcp-bridge`. The service unit uses absolute
paths under `/home/azureuser`; change them if your home directory differs.

1. Install the gateway:

   ```sh
   mkdir -p ~/.local/share/forward-flow-mcp-bridge/runtime
   cp runtime/package.json runtime/package-lock.json ~/.local/share/forward-flow-mcp-bridge/runtime/
   npm ci --prefix ~/.local/share/forward-flow-mcp-bridge/runtime
   ```

2. Install the script and the service unit:

   ```sh
   install -m 700 bin/forward-flow-mcp-bridge ~/.local/bin/
   cp systemd/forward-flow-mcp-bridge.service ~/.config/systemd/user/
   systemctl --user daemon-reload
   ```

3. Generate the config and start the service:

   ```sh
   forward-flow-mcp-bridge refresh
   systemctl --user enable --now forward-flow-mcp-bridge.service
   ```

4. Install the Caddy config. Edit the `@mcp` path list to match your server names first.

   ```sh
   sudo cp Caddyfile /etc/caddy/Caddyfile
   sudo systemctl reload caddy
   ```

Ports 80 and 443 must be reachable for certificate issuance.

## The bridge script

| Command | Effect |
|---|---|
| `refresh` | Rebuilds the gateway config from your MCP server definitions |
| `start`, `stop`, `restart`, `status` | Passed to `systemctl --user` for the service |

`refresh` reads command-based MCP servers from `~/.config/mcp-sync/global.json`
and `~/.claude.json`, then writes three files to `~/.config/forward-flow-mcp-bridge/`:

- `token`: a random API key, created once
- `servers.json`: the supergateway config
- `forward-flow-config.json`: client entries with each server's URL and the token

All three contain credentials. Do not commit them.

Server names must be at most 18 characters, using only ASCII letters, digits and `-`.
Run `refresh` then `restart` after changing your MCP servers.

## Authentication

Clients must send the key from the `token` file as `Authorization: Bearer <key>`
or `X-API-Key: <key>`. Caddy adds no authentication of its own.

## Notes

- `flush_interval -1` disables response buffering so streamed MCP responses are delivered immediately.
- `disable_tlsalpn_challenge` makes Caddy use the HTTP-01 challenge on port 80.
- Tested with Caddy 2.6.2.
