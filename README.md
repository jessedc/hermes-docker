# Hermes Agent on Synology DSM 7.2.2

> **Moved.** This repo now lives in the private [nas-docker](https://ds923.tail87aa42.ts.net:3000/jesse/nas-docker) repo, under
> `services/hermes/`, with its full history. Make changes there: this copy is no
> longer updated.

Runs the [Hermes Agent](https://hermes-agent.nousresearch.com/) container under
Synology **Container Manager**, with inference served by an OpenAI-compatible
endpoint on another machine on your tailnet.

```
   ┌──────────────── tailnet ────────────────┐
   │                                         │
   │   Synology NAS                          │
   │   ┌───────────────────────┐             │
   │   │ container: hermes     │             │      DGX Spark / GPU box
   │   │  gateway + dashboard  │──── HTTP ──────►  llama.cpp / vLLM /
   │   │  :9119 dashboard      │  /v1/chat/  │     LiteLLM
   │   │  :8642 API (optional) │  completions│     :8080
   │   └──────────┬────────────┘             │
   │              │ bind mount               │
   │   /volume1/docker/hermes/data           │
   └─────────────────────────────────────────┘
```

Everything Hermes owns lives in one bind-mounted folder. The image is
stateless, so upgrading is just a re-pull.

## What's here

| File                  | Goes where                                    |
| --------------------- | --------------------------------------------- |
| `docker-compose.yml`  | project folder (`/volume1/docker/hermes/`)    |
| `.env.example`        | copy to `.env` in the project folder          |
| `config.yaml.example` | copy to `config.yaml` in the **data** folder  |
| `minimal/`            | a stripped-down alternative to all three      |

Two deployments live here. The root files are the **full** reference: every
toolset on, dashboard published, the shape this repo started with. `minimal/`
is three capabilities and nothing else — local model, one MCP server, Telegram
— and is the better place to start. See [Minimal deployment](#minimal-deployment).

## Minimal deployment

`minimal/` runs three capabilities and nothing else: **the local model, one MCP
server, and Telegram.** Everything else is off and expands deliberately.

The case for starting here rather than trimming later, measured with
`hermes prompt-size` on both:

| | full | `minimal/` |
| --- | --- | --- |
| System prompt | 22,735 B | 10,534 B |
| Tool schemas | 55,460 B (21 tools) | 6,625 B (3 tools) |
| Published ports | 9119 | **none** |
| MCP tools exposed | 8 | 1 |

That ~78 KB → ~17 KB is prepended to *every* request. And with the dashboard
off there is no `ports:` key at all — nothing listens. Every connection is
outbound: the inference server over the tailnet, `api.telegram.org` over the
internet, and the MCP server sideways over a Docker network.

Telegram makes that possible. Hermes long-polls `getUpdates` rather than
registering a webhook (no `set_webhook` call exists in the adapter), so a chat
interface costs you no inbound port, no reverse proxy and no public DNS.

### Setting it up

Folders and Container Manager choreography are the same as the full path —
[step 2](#2-create-the-folders) and [step 6](#6-create-the-project-in-container-manager)
— with `minimal/`'s three files in place of the root ones:

```bash
scp minimal/docker-compose.yml minimal/.env.example you@nas:/volume1/docker/hermes/
scp minimal/config.yaml.example you@nas:/volume1/docker/hermes/data/
```

Rename to `.env` and `config.yaml`, `chmod 600 .env`, then fill in `.env`.

`config.yaml` and `docker-compose.yml` are **byte-identical on the Mac and the
NAS** — every host-specific value is a `${VAR}` resolved from the container
environment, and those placeholders survive the machine-rewrite that strips
comments. `.env` is the only file that differs.

For Telegram you need two values: a bot token from
[@BotFather](https://t.me/BotFather), and your numeric user ID from
`@userinfobot` for `TELEGRAM_ALLOWED_USERS`. Setting the token is all it takes
to enable the platform — there is no `config.yaml` key for it, and no
`hermes gateway setup` run required.

One token drives one poller. Starting a second gateway on the same token earns
a 409 Conflict from Telegram, so the Mac and the NAS need separate bots if you
run both.

### Verifying

```bash
sudo docker compose up -d
sudo docker ps --filter name=hermes          # healthy, PORTS column empty
sudo docker exec hermes hermes tools list    # memory, clarify, todo only
sudo docker exec hermes hermes prompt-size   # ~10.5 KB system, ~6.6 KB tools
sudo docker exec hermes hermes -z 'Reply with exactly: MINIMAL OK'
```

A placeholder bot token is caught at startup with a named error rather than a
confusing auth failure later:

```
ERROR gateway.config: telegram is enabled but TELEGRAM_BOT_TOKEN is set to a
placeholder value. The adapter will NOT be started.
```

### How the restriction actually works

`platform_toolsets` is an allowlist, per platform — only what is named there is
enabled. Add to **both** `cli` and `telegram` or a capability shows up in one
surface and not the other.

The non-obvious half is `known_builtin_toolsets`. Hermes treats a toolset it
has never seen as new and enables it by default, so a toolset simply *absent*
from the allowlist can still come up on — observed with `bfl` on a first run.
Seeding the known set closes that door, and stops an image update that adds
toolsets from silently re-expanding the agent. After a major image update,
re-check with `hermes tools list`.

MCP tools are allowlisted separately. `prompt-size` doesn't count them —
which is an accounting gap, not a free lunch: they are on the wire like
everything else. See [Expanding](#expanding) for what they actually cost.

```yaml
mcp_servers:
  brave-search:
    tools:
      include: [brave_web_search]     # exact names or globs
```

### What's on out of the box

Three toolsets, chosen on measured cost per unit of behaviour change rather
than on capability:

- **`memory`** (2,833 B) — without it the bot isn't worth talking to twice.
- **`clarify`** (2,414 B) — the agent can ask instead of guessing. Telegram
  renders it as inline buttons, and with `max_turns: 40` one round-trip is
  cheaper than a confident wrong answer 40 turns deep.
- **`todo`** (1,372 B) — a visible plan, which is the closest thing to
  scaffolding you get from a model doing shallow chain-of-thought.

`clarify` and `todo` add **nothing** to the system prompt. That's the whole
argument for them: they're the only two entries in the catalogue that change
how the agent behaves without also enlarging the always-on prefix.

`clarify` needs a surface that can answer it. The Telegram adapter implements
`send_clarify`; `hermes -z` has no callback, so there the tool returns an error
rather than hanging a scripted run. `agent.clarify_timeout` (default 3600 s,
1800 in `minimal/`) bounds how long a session thread waits for a tap.

### Reasoning

`agent.reasoning_effort` is **deliberately unset** in `minimal/`, and that is
not the same as `none`.

With `provider: custom`, Hermes' `CustomProfile` emits the level as a
*top-level* `reasoning_effort` field (not `extra_body.reasoning` — that's the
OpenRouter shape, and `_supports_reasoning_extra_body()` returns `False` for
any non-OpenRouter base URL). What the endpoint does with it is a separate
question, and on llama.cpp the answer is mostly "nothing".

llama.cpp will render a prompt without running the model, so you can settle
this for free. Note `/apply-template` sits at the root, not under `/v1`:

```bash
U=http://100.x.y.z:8080/apply-template
M='"messages":[{"role":"user","content":"hi"}]'
curl -s $U -d "{$M}"                             | jq -r .prompt | wc -c
curl -s $U -d "{$M,\"reasoning_effort\":\"high\"}" | jq -r .prompt | wc -c
```

Against `b1-ba360efe1` with DeepSeek V4 Flash:

| added to the request | rendered prompt |
| --- | ---: |
| *(nothing — baseline)* | 68 |
| `reasoning_effort: "high"` | 68 |
| `reasoning_effort: "medium"` | 68 |
| `reasoning_effort: "none"` | 69 |
| `chat_template_kwargs: {thinking: true, reasoning_effort: "high"}` | 544 |
| `chat_template_kwargs: {…, reasoning_effort: "max"}` | 596 |
| `chat_template_kwargs: {…, reasoning_effort: "medium"}` | 68 |

An unchanged byte count means the field never reached the template.

Two conclusions. The top-level field is ignored for every value except
`"none"` — that single byte is llama.cpp special-casing it into thinking-off,
which makes `none` a real switch and every positive level a dead letter. And
`medium` doesn't exist in this model: the Unsloth DeepSeek-V4-Flash template branches only on
`'high'` and `'max'`, so anything else injects no text at all.

Thinking is already on by default (llama.cpp passes `enable_thinking=true`,
and the template falls back to it), so **omitting the key is the middle
setting.** Setting `medium` lands in the same place, but by accident — it
reads like a connected dial and isn't one.

To actually raise it, use the channel that works. `providers.<name>.extra_body`
merges into the request body (verified: it merges with the profile's own
entries rather than replacing them):

```yaml
providers:
  spark:
    base_url: "${LLM_BASE_URL}"
    extra_body:
      chat_template_kwargs: {thinking: true, reasoning_effort: high}
```

Weigh it first. `total_slots: 1` means thinking tokens serialise across every
Telegram turn; the server runs `reasoning_format: "none"`, so `<think>` comes
back inline in `content` (Hermes strips it, but the tokens are real and count
against `compression.threshold: 0.50`); and `high` injects "absolute maximum
with no shortcuts permitted" into a 2.7-bpw quant already capped at
`max_turns: 40`. Don't combine it with `reasoning_effort: none` — the two
contradict each other on the wire.

### Expanding

The cost of every remaining toolset, measured one at a time on
`hermes-agent:0.20.0` with `hermes prompt-size`. **System** is what the toolset
adds to the *system prompt* on top of its schemas — the column that's easy to
forget, and the reason `skills` is nowhere near as cheap as its three tools
suggest:

| Toolset | Tools | Schema | System |
| --- | ---: | ---: | ---: |
| `code_execution` | 1 | 1,347 B | — |
| `vision` | 1 | 1,360 B | — |
| `todo` ✓ | 1 | 1,372 B | — |
| `tts` | 1 | 1,868 B | — |
| `web` | 2 | 1,912 B | — |
| `clarify` ✓ | 1 | 2,414 B | — |
| `memory` ✓ | 1 | 2,833 B | 1,442 B |
| `delegation` | 1 | 4,485 B | — |
| `terminal` | 2 | 4,821 B | — |
| `skills` | 3 | 5,621 B | **9,818 B** |
| `session_search` | 1 | 6,457 B | 188 B |
| `file` | 4 | 6,670 B | — |
| `cronjob` | 1 | 8,916 B | — |
| `computer_use` | 1 | 9,699 B | 5,244 B |

✓ = already on in `minimal/`. The *first* toolset you enable also adds a
one-off ~2,492 B of tool-usage guidance to the system prompt; `minimal/` has
already paid that. `browser` isn't listed because its ten tools sit behind a
Chromium check and ship zero schemas until you install it. Neither do `bfl`,
`image_gen`, `stt`, `video_gen`, `x_search`, `spotify`, `homeassistant`,
`discord`, `discord_admin` or `context_engine` — they're gated on credentials
or config this deployment doesn't have, so enabling them is inert rather than
expensive.

Roughly cheapest-and-safest first. Each step applies to both surfaces:

```bash
hermes tools enable file --platform cli
hermes tools enable file --platform telegram
```

1. more Brave tools — `brave_news_search`, `brave_summarizer`, via
   `tools.include`. Nearly free, though not for the reason `prompt-size`
   implies. MCP schemas *are* sent on every request; `prompt-size` just
   doesn't count them. What makes them cheap is `tools.tool_search`, which
   defaults to `auto`: the moment any MCP tool exists they all hide behind
   three bridge tools (`tool_search`, `tool_describe`, `tool_call`) plus a
   name-and-description listing, and full schemas are fetched on demand.
   Measured on the wire against Brave's real schemas:

   | Visible tool array | `tool_search: off` | default `auto` |
   | --- | ---: | ---: |
   | core three only | 6,625 B | 6,625 B |
   | + `brave_web_search` | 12,095 B | 8,599 B |
   | + all 8 Brave tools | 41,598 B | 9,233 B |

   The first Brave tool costs 1,974 B; the other seven cost 634 B between
   them. Turn `tool_search` off and the same eight cost 41,598 B.
2. `file` — read/write inside `/opt/data`. The first toolset that makes the
   agent useful for something other than talking.
3. `session_search` — 6,457 B for one tool. Wait until there's history worth
   searching; on day one it's the worst byte-for-byte deal on the list.
4. `terminal` + `code_execution` — real agent powers, and `code_execution` is
   the cheapest entry in the whole table. Blast radius is `/opt/data` unless
   you mount more.
5. `skills` — the 9,818 B is the always-on index, so prune `data/skills` to
   what you'll actually invoke *before* enabling, not after.
6. `browser` — Chromium in the container; raise `MEM_LIMIT` and add
   `shm_size: 1gb`.
7. the dashboard — set `HERMES_DASHBOARD: "1"`, add the three
   `HERMES_DASHBOARD_BASIC_AUTH_*` vars and a `ports:` entry. This is the step
   that gives the deployment its first listener; bind it to the NAS's tailnet
   address, not `0.0.0.0`.
8. `cronjob` — last, and 8,916 B for a single tool. Unattended turns with
   whatever you enabled above, and the reason `TELEGRAM_HOME_CHANNEL` exists.

Not worth enabling on a headless NAS at all: `computer_use` (9,699 B plus
5,244 B of system prompt, the most expensive toolset there is, and there's no
desktop), `vision` (the model is text-only), `tts`, and `image_gen`.
`delegation` is actively counterproductive against a llama.cpp server running
`total_slots: 1` — sub-agents serialise behind each other instead of running
in parallel.

## Prerequisites

- DSM 7.2.2 with **Container Manager** installed (Package Center).
- SSH enabled on the NAS: _Control Panel → Terminal & SNMP → Enable SSH_. You
  need it once for setup and it's the only comfortable way to use the Hermes
  CLI later.
- The **Tailscale** package installed on the NAS and logged in, or subnet
  routing to the inference host. Containers in bridge mode reach `100.x.y.z`
  addresses through the host's routing table, so no special network mode is
  needed.
- A reachable OpenAI-compatible server, and its model id.
- ~4 GB RAM free. Hermes itself is light; the model isn't running here.

`nousresearch/hermes-agent:latest` publishes `linux/amd64` and `linux/arm64`,
so both Intel and ARM Synology models that can run Container Manager are
covered.

## 1. Verify the endpoint from the NAS first

Do this before touching Container Manager. If it fails here, nothing else will
work, and you'll be debugging the wrong layer.

SSH into the NAS and run:

```bash
curl -s http://100.x.y.z:8000/v1/models | head -40
```

You want a JSON body with a `data` array. Note the `id` field — that string is
what goes into `model:` in `config.yaml`, exactly as printed.

If your server requires a token:

```bash
curl -s -H "Authorization: Bearer YOUR_TOKEN" http://100.x.y.z:8000/v1/models
```

Two common failure modes:

- **Connection refused / no route** — the inference server is bound to
  `127.0.0.1` on its own host. Rebind it to `0.0.0.0` or to its Tailscale
  address, and check the tailnet ACLs allow the NAS to reach it.
- **404** — your base URL needs (or must not have) the `/v1` suffix. Whatever
  path makes `/models` work is what `base_url` should point at.

Use the raw `100.x.y.z` address, not a MagicDNS name. See
[Tailnet DNS](#tailnet-dns) for why.

## 2. Create the folders

Still over SSH:

```bash
sudo mkdir -p /volume1/docker/hermes/data
sudo chown -R "$(id -u):$(id -g)" /volume1/docker/hermes
id     # note the uid= and gid= values, you need them in a moment
```

Adjust `/volume1` if your shared folder lives on another volume. The `docker`
shared folder is created by Container Manager; if it doesn't exist, make it in
_Control Panel → Shared Folder_ first.

## 3. Drop the files in

Copy `docker-compose.yml` and `.env.example` into
`/volume1/docker/hermes/`, and `config.yaml.example` into
`/volume1/docker/hermes/data/`. File Station works, or `scp` from your Mac:

```bash
scp docker-compose.yml .env.example you@nas:/volume1/docker/hermes/
scp config.yaml.example you@nas:/volume1/docker/hermes/data/
```

Then rename them on the NAS. File Station struggles with dot-files, so do this
over SSH:

```bash
cd /volume1/docker/hermes
mv .env.example .env
mv data/config.yaml.example data/config.yaml
chmod 600 .env
```

## 4. Fill in `.env`

Edit `/volume1/docker/hermes/.env`:

- `PUID` / `PGID` — the `uid=` and `gid=` from step 2. **Don't guess these.**
  The container drops to a non-root user, and if it doesn't match the mount's
  owner it can't write to `/opt/data` and the gateway won't start.
- `DASHBOARD_USER` / `DASHBOARD_PASS` / `DASHBOARD_SECRET` — required. The
  dashboard is bound to `0.0.0.0`, and Hermes refuses to start a non-loopback
  dashboard with no auth provider configured. Generate the two secrets with
  `openssl rand -base64 24` and `openssl rand -hex 32`.
- `LLM_API_KEY` — the bearer token for your endpoint, or `none`.
- `HERMES_DATA` — only if your path differs from
  `/volume1/docker/hermes/data`.

Every one of these is referenced with `:?` in the compose file, so a missing
value fails the deploy with a named error rather than starting something
half-configured.

## 5. Point `config.yaml` at your endpoint

Edit `/volume1/docker/hermes/data/config.yaml` and set three things under
`model:`:

```yaml
model:
  provider: "custom"
  base_url: "http://100.x.y.z:8000/v1"
  default: "the-id-from-step-1"
  api_key: "${LLM_API_KEY}"
  context_length: 32768
```

`provider: custom` is what tells Hermes to treat `base_url` as a generic
OpenAI-compatible endpoint instead of routing through a known provider.

The model key is **`default`**, not `model`. Hermes accepts `model:` when
reading and the CLI works fine with it, but `default` is the canonical name it
persists and the one the dashboard's "Main Model" selector reads — seed
`model:` and the UI shows no main model chosen until you pick one by hand.

`api_key: "${LLM_API_KEY}"` is substituted from the container's environment,
which the compose file populates from your `.env`. Note that Hermes expands
`${VAR}` but **not** bare `$VAR`; an unset variable is left in place verbatim
and logged as a warning.

Set `context_length` to whatever you actually launched the inference server
with. Self-hosted servers often don't advertise it, and if Hermes guesses high
you'll get truncation errors mid-session instead of clean compaction.

Writing this file up front is what lets you skip `hermes setup`, the
interactive wizard that normally collects API keys on first run.

## 6. Create the project in Container Manager

1. Open **Container Manager → Project → Create**.
2. **Project name:** `hermes`
3. **Path:** browse to `/volume1/docker/hermes` (the folder, not `data`).
4. Container Manager detects the existing `docker-compose.yml` and offers to
   use it. Accept.
5. Click through the web-portal wizard without configuring anything — you
   don't need a reverse proxy entry yet.
6. **Done → Build.**

The first build pulls ~1–2 GB. Watch the log pane; the gateway logs its bind
addresses on startup.

## 7. Verify

```bash
sudo docker ps --filter name=hermes
sudo docker logs --tail 100 hermes
```

Look for the dashboard binding on `0.0.0.0:9119` and no errors about
`/opt/data` permissions. Then browse to:

```
http://<nas-ip>:9119
```

Despite the `BASIC_AUTH` env var names, the dashboard redirects to a `/login`
form rather than issuing an HTTP auth challenge. Sign in with `DASHBOARD_USER`
/ `DASHBOARD_PASS`.

Confirm Hermes resolved your endpoint rather than a default provider:

```bash
sudo docker exec -it hermes hermes config
```

Under `◆ Model` you should see your `base_url` and your model id as `default`.

Then smoke-test the whole chain non-interactively with `-z`:

```bash
sudo docker exec hermes hermes -z "What is 17 times 23? Answer with just the number."
```

A correct answer means container → endpoint → tool loop all work. If you
configured `auxiliary.vision`, check that path too by dropping a PNG in the
data folder and asking about it:

```bash
sudo docker exec hermes hermes -z "Describe the colours in /opt/data/test.png"
```

## Day-to-day use

The dashboard is the main interface. For the CLI, exec into the container —
Hermes automatically drops to its runtime user, so don't add `-u`:

```bash
sudo docker exec -it hermes hermes chat
sudo docker exec -it hermes hermes config edit
sudo docker exec -it hermes hermes -p work gateway start   # extra profile
```

Logs are tee'd to disk as well as stdout:

```
/volume1/docker/hermes/data/logs/gateways/default/current
```

### `config.yaml` is machine-managed

Changing anything in the dashboard, or running `hermes config set`, rewrites
`config.yaml` in place. The rewrite strips every comment, reorders and renames
keys to their canonical form, and stamps a `_config_version`. Keys the writer
doesn't round-trip are silently dropped — `model.context_length` is one, so it
disappears the first time you change the main model in the UI.

Treat the file as seed state rather than something you maintain by hand. After
any UI change, spot-check what survived:

```bash
docker exec hermes hermes config get model.context_length
```

Keep the annotated copy in `config.yaml.example` under version control; that's
the one with the reasoning in it.

## Optional: expose Hermes' own OpenAI-compatible API

Distinct from the upstream endpoint Hermes consumes — this makes *Hermes* look
like an OpenAI API to other tools on your tailnet.

1. Add a strong key to `.env`:

   ```bash
   echo "API_SERVER_KEY=$(openssl rand -hex 32)" >> /volume1/docker/hermes/.env
   ```

2. In `docker-compose.yml`, set `API_SERVER_ENABLED: "true"` and uncomment the
   `API_SERVER_HOST`, `API_SERVER_PORT`, and `API_SERVER_KEY` lines, plus the
   `8642` port mapping.
3. Rebuild the project in Container Manager.

Hermes requires the key to be at least 8 characters whenever the API server is
bound to `0.0.0.0`, and refuses to start otherwise. Health check:

```bash
sudo docker exec hermes curl -s http://127.0.0.1:8642/health
```

## MCP servers

MCP servers add tool sets to the agent. Add one with the CLI rather than by
editing `config.yaml` — it probes the endpoint, prints the tools it discovered,
and only writes the entry if the connection succeeded:

```bash
sudo docker exec -it hermes hermes mcp add brave-search --url <endpoint>
sudo docker exec hermes hermes mcp list
sudo docker exec hermes hermes mcp test brave-search
```

`hermes mcp add` is interactive — it asks whether the server needs auth, then
whether to enable all discovered tools. Without a TTY it reaches the second
prompt and prints `Cancelled.` without saving, so use `-it`, or pipe the
answers (`printf 'n\ny\n' | docker exec -i ...`).

Tools land in the next session, not the running one.

### Reaching an MCP server on the NAS itself

This is the case that looks trivial and isn't. If the MCP server is another
container on the same NAS, published to loopback and exposed to the tailnet
with `tailscale serve`, then the obvious URL — the one that works from every
other machine on your tailnet — is the one URL that cannot work from here.

Two independent reasons:

1. **MagicDNS doesn't resolve in the container.** The same limitation as
   [Tailnet DNS](#tailnet-dns) below.
2. **`tailscale serve` only accepts connections from *other* tailnet peers.**
   Not from the machine doing the serving, and not from its containers. This
   is worth stating plainly because it is easy to misdiagnose as a firewall or
   a certificate problem. Verified from a shell on the NAS itself, with DNS
   taken out of the picture:

   ```bash
   curl --resolve ds923.tail87aa42.ts.net:8422:100.83.84.78 \
        https://ds923.tail87aa42.ts.net:8422/mcp        # → 000, no connection
   curl http://127.0.0.1:8422/mcp                       # → 406, server is fine
   ```

   The same request from any other tailnet node succeeds. The container case
   was not tested end to end, but it follows: a bridge-mode container's traffic
   is host-originated and sources from `172.x.y.z`, so it is on the failing
   side of that line for the same reason the host shell is.

Pinning the name with `extra_hosts` fixes only reason 1 and leaves reason 2
intact. Falling back to `https://100.x.y.z:8422/mcp` fails TLS as well — the
Tailscale-issued certificate carries `DNS:ds923.tail87aa42.ts.net` as its only
SAN, and no IP SAN.

The fix is to stop going out to the tailnet and back. Both containers are on
the same Docker host, so put them on the same Docker network and address the
server by container name **on its internal port** — not the host port it
publishes:

```yaml
# docker-compose.yml
services:
  hermes:
    networks:
      - default
      - brave-search-mcp_default

networks:
  default:
  brave-search-mcp_default:
    external: true
```

```bash
sudo docker network ls | grep brave      # confirm the real network name
sudo docker exec -it hermes hermes mcp add brave-search \
     --url http://brave-search-mcp:8080/mcp
```

Note `8080`, the port inside the MCP container, not the `8422` it publishes on
the NAS's loopback. Compose names the default network `<project>_default`, so
a project folder of `brave-search-mcp` yields `brave-search-mcp_default`;
`external: true` attaches to it instead of creating it, which means that
project has to be up first.

This route is also strictly better than the tailnet one: no DNS, no TLS
handshake, no dependency on Tailscale being up for a call between two
containers a bridge apart.

## Networking

### Tailnet DNS

The Synology Tailscale package puts a `tailscale0` interface on the host, so
bridge-mode containers reach `100.x.y.z` addresses fine — the host routes for
them. **MagicDNS names are a different story:** the package doesn't rewrite
container resolvers, so `llm.your-tailnet.ts.net` typically won't resolve
inside the container even though it resolves on the NAS shell.

Use the raw `100.x.y.z` address. If you'd rather use the name, pin it in
`docker-compose.yml`:

```yaml
extra_hosts:
  - "llm.your-tailnet.ts.net:100.x.y.z"
```

Pointing the container's resolver at `100.100.100.100` also works, but it
routes *all* container DNS through Tailscale, which breaks name resolution if
Tailscale is down. The `extra_hosts` pin is the more predictable option.

Tailscale addresses are stable per node, so `extra_hosts` won't drift. If you
rebuild the inference host from scratch it'll get a new address — update both
`config.yaml` and `extra_hosts` then.

### Locking the ports down

By default the dashboard is published on every NAS interface. To make it
tailnet-only, put the NAS's own Tailscale address in `.env`:

```bash
BIND_ADDR=100.a.b.c
```

and rebuild. Docker then binds the published port to that interface only.
Note this hard-codes the NAS's tailnet address into the deployment — if it
ever changes, the container will fail to start with a bind error, which is a
loud enough failure to be fine.

The DSM firewall (_Control Panel → Security → Firewall_) is a reasonable
second layer if you'd rather keep LAN access but restrict it by source subnet.

### Reverse proxy

If you want HTTPS, _Control Panel → Login Portal → Advanced → Reverse Proxy_
can front `localhost:9119` with a DSM certificate. Basic auth passes through
unchanged.

## Updating

Docker installs don't support `hermes update` — you update the image:

```bash
cd /volume1/docker/hermes
sudo docker compose pull
sudo docker compose up -d
```

Or in Container Manager: select the project → **Action → Build**, which
re-pulls `:latest`. State in `data/` is untouched. Pin to a specific tag
instead of `latest` in `docker-compose.yml` if you'd rather control when
versions move.

## Backup

Everything that matters is one folder:

```bash
sudo tar czf /volume1/backups/hermes-$(date +%F).tar.gz \
  -C /volume1/docker/hermes data .env
```

That covers `config.yaml`, `.env`, `SOUL.md`, sessions, memories, skills, and
cron jobs. Add `/volume1/docker/hermes` to Hyper Backup for something
scheduled.

## Troubleshooting

**Container Manager rejects `deploy:`** — some DSM builds are fussy about it.
Replace the whole `deploy:` block with the legacy equivalent:

```yaml
    mem_limit: 4g
```

Memory only — Synology can't cap CPU.

**Permission errors on `/opt/data`, or the gateway exits immediately** —
`PUID`/`PGID` don't match the folder owner. Confirm with:

```bash
stat -c '%u %g' /volume1/docker/hermes/data
```

and make `.env` agree, or re-run the `chown` from step 2.

**Dashboard container starts then dies** — Hermes fails closed when the
dashboard binds to a non-loopback address with no auth provider. Check that
`DASHBOARD_USER` and `DASHBOARD_PASS` actually made it in:

```bash
sudo docker exec hermes env | grep DASHBOARD
```

`HERMES_DASHBOARD_INSECURE=1` is deprecated and ignored, so it isn't a way
around this.

**Model calls fail, dashboard is fine** — the endpoint isn't reachable from
*inside* the container, even if it was from the NAS shell:

```bash
sudo docker exec hermes curl -s http://100.x.y.z:8000/v1/models
```

Almost always DNS (see above) or a tailnet ACL.

**Truncation or context errors mid-session** — `context_length` in
`config.yaml` is larger than what the inference server was launched with.

**Deploy fails with `variable is not set`** — a `:?` guard fired. The message
names the missing variable; add it to `.env`. Also check the `.env` is in the
same folder as `docker-compose.yml`, not in `data/`.

## Security notes

- The dashboard's basic auth is the only thing between your LAN and an agent
  that can run shell commands. Use a long random password, and prefer
  `BIND_ADDR=100.a.b.c` so it isn't exposed to the LAN at all.
- `.env` holds secrets in plaintext — `chmod 600` it, and don't let the
  `docker` shared folder get indexed or synced anywhere.
- The Docker socket mount is commented out deliberately. Mounting it gives the
  agent effective root on the NAS, including every other container.
- Shell commands from the agent run inside this container as `PUID`, and see
  only `/opt/data` unless you mount more. Adding host paths as volumes widens
  that blast radius.
- Don't forward port 9119 through your router. Reach it over the tailnet.

## Appendix: running on macOS

The compose file is plain Compose v2 with nothing Synology-specific in it, and
the image publishes `linux/arm64`, so the same stack runs under Docker Desktop.
Only the values in `.env` change — steps 2, 3 and 6 above are Container Manager
choreography you can skip in favour of `docker compose up -d`.

```bash
HERMES_DATA=/Users/you/Development/hermes-docker/data
PUID=501            # your uid; largely cosmetic under Docker Desktop, which
PGID=20             # remaps bind-mount ownership via virtiofs
BIND_ADDR=127.0.0.1 # dashboard on loopback only
MEM_LIMIT=2G        # must fit inside Docker Desktop's VM allocation
```

`MEM_LIMIT` is the one that bites: Compose will happily accept a limit larger
than the VM, and the container gets OOM-killed under load instead of failing at
start.

### Pointing at the DGX Spark over the tailnet

This is the setup actually in use: Hermes under Docker Desktop on the Mac, with
inference on a DGX Spark (`edgexpert-7b1e`, NVIDIA GB10) running llama.cpp on
port 8080, reached over the tailnet.

Docker Desktop routes container traffic to `100.x.y.z` addresses through the
host, so this needs no `host.docker.internal` indirection and no special network
mode. Prove that from inside a container before configuring anything, because a
working `curl` on the Mac does not prove the container can get there:

```bash
docker run --rm curlimages/curl -s -o /dev/null -w '%{http_code}\n' \
  http://100.x.y.z:8080/v1/models          # want 200
```

Then ask the server what it is. `/v1/models` gives the id and the context window
it was launched with; `/props` gives the modalities:

```bash
curl -s http://100.x.y.z:8080/v1/models | jq '.data[] | {id, n_ctx: .meta.n_ctx}'
curl -s http://100.x.y.z:8080/props    | jq '.modalities'
```

The Spark serves one model at a time, so `config.yaml` follows whatever is
loaded there. Currently that is DeepSeek V4 Flash, matching the model the other
tools on this tailnet use:

```yaml
model:
  provider: "custom"
  base_url: "http://100.x.y.z:8080/v1"
  default: "DeepSeek-V4-Flash-0731-UD-IQ2_M"
  api_key: "${LLM_API_KEY}"
  context_length: 262144

auxiliary:
  vision:
    provider: "auto"
```

llama.cpp reports the model id as the full HF cache path, and ignores the
`model` field in a request entirely when it has one model loaded — so the short
alias above works and stays readable. That is a llama.cpp affordance, not a
general one: vLLM and LiteLLM both validate the id against what `/v1/models`
prints.

Vision depends on the loaded model, so it is worth re-checking on every swap.
DeepSeek V4 Flash is text-only, and `provider: "auto"` is the honest setting
while it is loaded — there is nothing to route an image to. With a multimodal
GGUF loaded, `/props` reports `{"vision": true, ...}` and the vision block is
better spelled out explicitly:

```yaml
auxiliary:
  vision:
    provider: "custom"
    model: "unsloth/Muse-Glimmer-30B-GGUF:UD-Q6_K_XL"
    base_url: "http://100.x.y.z:8080/v1"
    api_key: "${LLM_API_KEY}"
    timeout: 180
```

`auto` would reuse the main model, which is the right endpoint, but it depends
on Hermes deciding that model is vision-capable — and a locally-served GGUF
carries no capability metadata for it to consult. Being explicit removes the
guess. The other `auxiliary` slots can stay on `auto`; they follow the main
model.

llama.cpp only checks the bearer token when started with `--api-key`, so
`LLM_API_KEY=none` is an ignored placeholder here. Keep it non-empty regardless
— an empty key can trip client-side validation before a request is even sent.

Take `context_length` from the server's own `n_ctx`, not the model's
architectural maximum. Hermes compacts at 50% of this value, so an inflated
number means sessions balloon before anything trims them.

Verify through the container rather than against the server:

```bash
docker exec hermes hermes -z 'Reply with exactly: SPARK OK'
```

And when a multimodal model is loaded, the vision path too:

```bash
docker exec hermes hermes -z 'Use your vision capability on /opt/data/workspace/test.png. Reply with only the shape and its colour.'
```

Use an image whose answer you already know — a flat coloured shape on white is
enough to tell a working vision path from a plausible hallucination.

One operational note: llama.cpp serves `total_slots: 1` unless told otherwise,
so requests queue instead of running in parallel. A vision call landing during a
long generation waits its turn, which reads as a hang if you aren't expecting
it. Check what the server is chewing on with:

```bash
curl -s http://100.x.y.z:8080/slots | jq '.[0] | {is_processing, n_prompt_tokens}'
```

### MCP servers from the Mac

The tailnet URL that [cannot work from the NAS](#reaching-an-mcp-server-on-the-nas-itself)
is the correct one here, because the Mac genuinely is a different tailnet peer
from the NAS hosting the server:

```bash
docker exec -it hermes hermes mcp add brave-search \
    --url https://ds923.tail87aa42.ts.net:8422/mcp
```

Docker Desktop routes container traffic to `100.x.y.z` through the host and does
rewrite container DNS, so both the MagicDNS name and the certificate work
without an `extra_hosts` pin. No auth — the server sits behind `tailscale serve`
and is only reachable from the tailnet in the first place.

### Moving this to the NAS

The two deployments share a compose file, so porting is mostly a matter of
knowing what is genuinely host-specific.

Transfers untouched: the whole `model:` block (`base_url` is already a raw
Tailscale IP, which is what Synology requires, and the Spark is a third node to
both machines), plus `agent:`, `terminal:`, `compression:`, `memory:` and
`updates:`. So does `web: backend: brave-free` with its `BRAVE_SEARCH_API_KEY`
— that is the Brave HTTP API, unrelated to the MCP server.

Needs a per-host value, all of them already `.env` variables: `PUID`/`PGID`,
`HERMES_DATA`, `BIND_ADDR`, `MEM_LIMIT`. Generate fresh dashboard credentials
rather than copying them; on the Mac the dashboard is on loopback, on the NAS
it is exposed.

Needs rewriting: the `mcp_servers:` entry, for the reasons in
[MCP servers](#reaching-an-mcp-server-on-the-nas-itself).

**Do not copy most of `data/`.** The image is multi-arch, and the two hosts are
not the same architecture — a DS923+ is `x86_64` while an Apple Silicon Mac runs
the `linux/arm64` image. Anything Hermes compiled or downloaded for one is wrong
on the other:

| Path | Why |
| --- | --- |
| `lazy-packages/` (~300 MB) | native wheels, e.g. `*.cpython-313-aarch64-linux-gnu.so` |
| `bin/tirith` | arch-specific binary |
| `.local/`, `cache/`, `sandboxes/` | same problem |
| `gateway.pid`, `*.lock`, `.hermes_history` | run state from the other host |

Hermes rebuilds all of that on first start. What is worth carrying over is
`config.yaml`, `SOUL.md`, `memories/`, and any skills you wrote. `state.db`,
`kanban.db` and `projects.db` are SQLite and portable across architectures, but
copy them only with the container stopped — otherwise you will take a snapshot
with an uncheckpointed `-wal` file beside it.

One setting to reconsider on the way over: `browser.backend: browser-use` wants
`shm_size: 1gb` and a heavy dependency tree, which fits poorly under a NAS-sized
`MEM_LIMIT`.

## Reference

- [Hermes Docker guide](https://hermes-agent.nousresearch.com/docs/user-guide/docker/)
- [Configuration reference](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)
- [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security)
