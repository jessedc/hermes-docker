# Hermes Agent — minimal deployment

Local model + one MCP server + Telegram. Nothing else enabled.

No published ports: the dashboard is off and Telegram long-polls, so every
connection is outbound. Identical files on macOS and Synology — only `.env`
differs.

For the reasoning behind any of this, and how to turn things back on, see
[Minimal deployment](../README.md#minimal-deployment) in the parent README.

## You need

- Docker Desktop, or DSM 7.2.2 with Container Manager.
- An OpenAI-compatible endpoint reachable from the host, and its model id.
- A Telegram bot token from [@BotFather](https://t.me/BotFather), and your
  numeric user ID from `@userinfobot`.
- An MCP server URL (optional — to skip it, delete the `mcp_servers` block
  from `config.yaml` *and* the `BRAVE_MCP_URL` line from `docker-compose.yml`).

## 1. Folders

```bash
mkdir -p /volume1/docker/hermes/data          # NAS; adjust for macOS
chown -R "$(id -u):$(id -g)" /volume1/docker/hermes
id                                            # note uid= and gid=
```

## 2. Files

Copy into the project folder, `config.yaml.example` into `data/`:

```bash
scp docker-compose.yml .env.example you@nas:/volume1/docker/hermes/
scp config.yaml.example you@nas:/volume1/docker/hermes/data/
```

Rename on the host (File Station struggles with dot-files, use SSH):

```bash
cd /volume1/docker/hermes
mv .env.example .env && chmod 600 .env
mv data/config.yaml.example data/config.yaml
```

## 3. Fill in `.env`

| Variable | Notes |
| --- | --- |
| `PUID` / `PGID` | from `id` in step 1. Don't guess — wrong values mean the container can't write `/opt/data`. |
| `HERMES_DATA` | absolute path to the `data` folder |
| `LLM_BASE_URL` | raw `100.x.y.z` address, not a MagicDNS name. Usually ends `/v1`. |
| `LLM_MODEL` | as `/v1/models` reports it (llama.cpp accepts a short alias; vLLM does not) |
| `TELEGRAM_BOT_TOKEN` | from @BotFather. One token = one gateway. |
| `TELEGRAM_ALLOWED_USERS` | comma-separated numeric IDs. Required. |
| `BRAVE_MCP_URL` | **the only genuinely host-dependent value** — see below |

### The MCP URL

**macOS** — a different tailnet peer, so the tailnet URL works as-is:

```bash
BRAVE_MCP_URL=https://ds923.tail87aa42.ts.net:8422/mcp
```

**NAS** — the MCP server is a container on this same host. The tailnet URL
*cannot* work here (`tailscale serve` only accepts other tailnet peers). Use
the container name and its **internal** port:

```bash
BRAVE_MCP_URL=http://brave-search-mcp:8080/mcp
```

and uncomment both `networks:` blocks in `docker-compose.yml`, after checking
the real network name:

```bash
sudo docker network ls | grep brave
```

The MCP project must be running first — the network is `external`.

## 4. Start

```bash
docker compose up -d
```

On DSM: **Container Manager → Project → Create**, path
`/volume1/docker/hermes`, accept the detected compose file, **Build**.

## 5. Verify

```bash
docker ps --filter name=hermes        # healthy, PORTS column empty
docker exec hermes hermes tools list  # only `memory` enabled
docker exec hermes hermes mcp test brave-search
docker exec hermes hermes -z 'Reply with exactly: MINIMAL OK'
```

Then message the bot from an allowlisted account.

## Troubleshooting

**`required variable X is missing a value`** — a `.env` value is unset.
Compose names the variable; it fails before starting anything.

**`TELEGRAM_BOT_TOKEN is set to a placeholder value`** — the token is a
sample-looking string. The gateway starts, the adapter doesn't. An *empty*
token trips Compose's guard above instead, before the container starts.

**`No messaging platforms enabled`** — no valid token. Everything else still
works; the CLI is reachable via `docker exec`.

**Container can't write `/opt/data`** — `PUID`/`PGID` don't match the folder
owner. Re-run `id` on the host.

**MCP won't connect from the NAS** — you're using the tailnet URL. See
[the MCP URL](#the-mcp-url).
