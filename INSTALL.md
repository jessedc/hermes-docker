# INSTALL — Hermes Agent on the Synology DS923+

A verified, step-by-step record of deploying [`minimal/`](minimal/) to the NAS
`ds923`, with inference on the DGX Spark and the Brave MCP server running as a
sibling container on the same host.

Every value below was **confirmed on the hardware**, not assumed. Where this
deployment departs from `minimal/README.md`, the deviation is marked **⚠** and
the evidence is given — three of them are host limitations that the generic
docs get wrong for this NAS.

---

## What this gives you

A capable assistant running entirely on hardware you own. The NAS holds the
agent and all of its state, a second machine on your LAN serves the model, and
the only traffic that leaves your network is Telegram. There is no model-vendor
API key, no cloud dependency, and — apart from a dashboard pinned to loopback —
nothing listening on your LAN. It runs on DSM 7.2.2 under Container Manager as
a normal project, so it starts with the NAS and is managed like anything else
there.

- **Chat from anywhere, via Telegram.** Reachable from your phone or desktop,
  restricted to an explicit allowlist of user IDs — the bot ignores everyone
  else. It long-polls rather than registering a webhook, so no inbound port, no
  reverse proxy, and no port forwarding.
- **A web dashboard** for sessions, settings, and model selection, behind a
  login. Published on loopback and reached over the tailnet, so it is never
  exposed to the LAN.
- **Your own model.** Any OpenAI-compatible endpoint. Conversations never reach
  a third party.
- **Memory that persists.** Facts and a user profile survive across sessions,
  which is what makes it worth talking to more than once.
- **It asks instead of guessing.** An under-specified request comes back as a
  question with tappable buttons rather than forty turns in the wrong
  direction. Questions time out after 30 minutes so an unanswered one doesn't
  hold a worker open.
- **It keeps a plan.** A running todo list the model maintains, which is most
  of what keeps a small local model on track.
- **A long context that manages itself.** A 262,144-token window, with history
  compressed automatically once it is half full while the twenty most recent
  messages are kept intact.
- **Bounded runs.** Forty turns per task by default, so a runaway loop stops on
  its own instead of grinding your inference server for an afternoon.
- **One folder holds everything.** All state lives in a single bind mount and
  the image is stateless, so upgrading is a re-pull and backing up is one
  `tar`. Take that backup yourself before upgrading: a new image snapshots
  `config.yaml` and `.env` only when it migrates the config schema, and nothing
  else. Hermes's own pre-update backup belongs to `hermes update`, which Docker
  installs never run.
- **Nothing turns itself on.** Capabilities are allowlisted per surface, and
  the known-toolset seed stops an image update from silently enabling new ones.

## The deployment

```
                     tailnet
                        │  https://ds923.tail87aa42.ts.net:9119
  ┌─────────────────────┼──────────┐        ┌────────────────────────────┐
  │ Synology DS923+     ▼          │        │ DGX Spark (edgexpert-7b1e) │
  │ 192.168.1.222  tailscale serve │        │ 192.168.1.50               │
  │                     │          │        │                            │
  │  ┌──────────────────▼───────┐  │  LAN   │   llama.cpp :8080          │
  │  │ hermes                   │──┼────────┼──►  /v1  (bound 0.0.0.0)   │
  │  │ dashboard 127.0.0.1:9119 │  │        │                            │
  │  └────────────┬─────────────┘  │        └────────────────────────────┘
  │               │ docker network │
  │  ┌────────────▼─────────────┐  │                    ▲
  │  │ brave-search-mcp :8080   │  │                    │ outbound only
  │  └──────────────────────────┘  │                    │
  └────────────────────────────────┘        api.telegram.org (long-poll)
```

One published port, on **loopback only**. The dashboard listens on
`127.0.0.1:9119` and `tailscale serve` fronts it for the tailnet — nothing is
exposed to the LAN and no port is forwarded. Everything else is outbound:
inference over the LAN, Telegram long-polling, and the MCP server sideways over
a Docker network.

## Verified values

| Setting | Value | How it was confirmed |
| --- | --- | --- |
| `PUID` / `PGID` | `1026` / `100` | `id` → `uid=1026(collis-admin) gid=100(users)` |
| `HERMES_DATA` | `/volume1/docker/hermes/data` | created in step 2 |
| `TZ` | `America/Los_Angeles` | NAS reports `PDT-0700`; IANA name so DST tracks |
| `LLM_BASE_URL` | `http://192.168.1.50:8080/v1` | **⚠ see Finding 1** |
| `LLM_MODEL` | `DeepSeek-V4-Flash-0731-UD-IQ2_M` | short alias accepted — a chat completion returned 200 |
| `LLM_API_KEY` | `none` | server runs without `--api-key` |
| `BRAVE_MCP_URL` | `http://brave-search-mcp:8080/mcp` | `docker ps` → `127.0.0.1:8422->8080/tcp` |
| MCP network | `brave-search-mcp_default` | `docker network ls` |
| `context_length` | `262144` | `/v1/models` → `meta.n_ctx` = 262144 |
| `BIND_ADDR` | `127.0.0.1` | dashboard on loopback; tailnet via `tailscale serve` |
| Dashboard | `:9119` | `https://ds923.tail87aa42.ts.net:9119` |

---

## Findings — where the generic docs are wrong for this NAS

### ⚠ Finding 1 — Tailscale runs in userspace mode; `100.x` is unroutable

`minimal/.env.example` instructs you to use the raw `100.x.y.z` tailnet
address. **On this NAS that cannot work.** The Synology Tailscale package is
running without a TUN device, so the host kernel has no route to the tailnet:

```console
$ ip addr show tailscale0
Device "tailscale0" does not exist.

$ ip route get 100.91.32.54
100.91.32.54 via 192.168.1.1 dev eth0  src 192.168.1.222

$ tailscale ping 100.91.32.54
pong from edgexpert-7b1e (100.91.32.54) via 192.168.1.50:41641 in 2ms
```

Read those together: the userspace WireGuard stack is **healthy** (`tailscale
ping` succeeds in 2 ms), but the kernel sends `100.x` to the LAN gateway, which
drops it. So `curl` times out with exit 28 while `tailscale ping` works.

This matters twice over: a bridge-mode container reaches `100.x` *only* through
the host routing table, so the container could never have reached it either.

**Resolution.** The NAS and the Spark are on the same physical LAN
(`192.168.1.0/24`), so the tailnet buys nothing between them — it would encrypt
a hop between two machines one switch apart. Use the LAN address. llama.cpp is
bound to `0.0.0.0`, verified with a 200 from both the NAS and the Mac.

> **Pin `192.168.1.50` as a DHCP reservation.** The tailnet IP was stable by
> nature; a LAN IP is not unless you make it so. A lease change silently breaks
> inference.

Note that `tailscale serve` *does* work on this NAS despite userspace mode —
it's how the Brave MCP is published on `:8422`. Only outbound kernel routing to
`100.x` is broken.

### ⚠ Finding 2 — `scp` fails; DSM's SFTP subsystem is off

```console
$ scp minimal/docker-compose.yml nas:/volume1/docker/hermes/
subsystem request failed on channel 0
scp: Connection closed
```

OpenSSH 9.0+ made `scp` use the SFTP protocol by default, and DSM ships with
the SFTP subsystem disabled. Force the legacy protocol with **`-O`**:

```bash
scp -O nas/docker-compose.yml nas/.env nas:/volume1/docker/hermes/
```

Fallback that needs no subsystem at all:

```bash
ssh nas 'cat > /volume1/docker/hermes/docker-compose.yml' < nas/docker-compose.yml
```

To fix permanently: _Control Panel → File Services → FTP → SFTP → Enable SFTP
service_. Not required for this deployment.

(The `WARNING: connection is not using a post-quantum key exchange algorithm`
banner is unrelated and cosmetic — DSM ships an older OpenSSH.)

### ⚠ Finding 3 — `cpus:` is rejected by the Synology kernel

```
Error response from daemon: NanoCPUs can not be set, as your kernel does not
support CPU CFS scheduler
```

DSM kernels are built without `CONFIG_CFS_BANDWIDTH`, so the CFS quota
controller does not exist and the daemon refuses to create the container.
**`deploy.resources.limits.cpus` cannot work on any Synology NAS.**

This was a genuine defect in this repo — `minimal/docker-compose.yml` claimed
in its header to be "identical on both" macOS and Synology while carrying a key
that only works on the Mac, and the README's troubleshooting section offered
the legacy `cpus: 2.0` as a *remedy*, which fails the same way.

**Fixed.** `cpus:` and `CPU_LIMIT` are gone from both compose files and both
`.env.example` files. Memory limits are unaffected. If you are working from an
older checkout, delete the `cpus:` line.

### Note — folder ACLs and `.env`

`/volume1/docker/hermes` inherits `everyone::allow:r-x` from `/volume1/docker`
with file-inherit set, so files created there are readable by any DSM user.
`chmod 600 .env` **does** strip the inherited ACL — the resulting file shows
`-rw-------` with no `+` marker. No further action needed, but the `chmod` is
load-bearing rather than hygiene, and it does **not** survive `scp`, so re-run
it after every copy.

The folder's ACL is otherwise byte-identical to sibling projects created by
Container Manager.

---

## Prerequisites

- DSM 7.2.2 with **Container Manager** installed.
- SSH enabled: _Control Panel → Terminal & SNMP → Enable SSH_.
- The inference server reachable from the NAS **on the LAN** (Finding 1).
- The Brave MCP project already running — its network is `external`, so it must
  exist before Hermes starts.
- A Telegram bot token from [@BotFather](https://t.me/BotFather) and your
  numeric ID from `@userinfobot`.

> **One token = one poller.** A second gateway on the same token gets `409
> Conflict` from Telegram. Use a **separate bot** from any instance running on
> the Mac.

---

## Step 1 — Pre-flight on the NAS

All read-only. Every value feeds `.env`.

```bash
ssh nas

# The endpoint, from the NAS itself. If this fails, nothing else matters.
curl -s --max-time 10 http://192.168.1.50:8080/v1/models | head -c 200; echo

# uid/gid → PUID/PGID. Do not guess these.
id

# The MCP container's name and its INTERNAL port
sudo docker ps --format '{{.Names}}\t{{.Image}}\t{{.Ports}}' | grep -i brave

# The Docker network to join
sudo docker network ls | grep -i brave
```

In the `docker ps` output — `127.0.0.1:8422->8080/tcp` — the **left** side is
the host port that `tailscale serve` fronts, unusable from a container. The
**right** side, `8080`, is what goes in `BRAVE_MCP_URL`.

If the `curl` produces nothing, run it as `curl -sS -v` — `-s` hides the error.
A timeout (exit 28) rather than a refusal points at Finding 1.

## Step 2 — Create the folders

```bash
sudo mkdir -p /volume1/docker/hermes/data
sudo chown -R "$(id -u):$(id -g)" /volume1/docker/hermes
ls -ld /volume1/docker/hermes /volume1/docker/hermes/data
```

Both must show `collis-admin users`. A mismatch against `PUID`/`PGID` is the
most common reason the container starts and then cannot write `/opt/data`.

## Step 3 — Stage the files on the Mac

Prepare them locally so the exact validated bytes reach the NAS.

```bash
cd ~/Development/hermes-docker
mkdir -p nas
cp minimal/config.yaml.example nas/config.yaml     # unchanged — portable by design
cp minimal/docker-compose.yml  nas/docker-compose.yml
cp minimal/.env.example        nas/.env
```

One edit to `nas/docker-compose.yml` — join the MCP container's network by
uncommenting both `networks:` blocks:

```yaml
    networks:
      - default
      - brave-search-mcp_default
```
```yaml
networks:
  default:
  brave-search-mcp_default:
    external: true
```

(Finding 3's `cpus:` fix is already in the repo, so there is nothing to do for
it here.)

Then fill in `nas/.env` with the verified values from the table above, plus the
two Telegram values. `config.yaml` needs **no changes** — every host-specific
value in it is a `${VAR}`, and `context_length: 262144` already matches the
server's reported `n_ctx`.

Validate before anything crosses the network — this catches an unsatisfied
`:?` guard or a YAML slip where it is cheap to fix:

```bash
cd nas && docker compose config >/dev/null && echo "compose OK"
shasum -a 256 docker-compose.yml config.yaml
```

## Step 4 — Copy to the NAS

```bash
cd ~/Development/hermes-docker
scp -O nas/docker-compose.yml nas/.env nas:/volume1/docker/hermes/
scp -O nas/config.yaml                 nas:/volume1/docker/hermes/data/
```

`-O` is required — see Finding 2. Then on the NAS:

```bash
cd /volume1/docker/hermes
chmod 600 .env                     # does NOT survive scp; re-run after every copy
sha256sum docker-compose.yml data/config.yaml
sudo docker compose config >/dev/null && echo "compose OK"
```

Compare the checksums against the Mac. Run `docker compose config` here even if
you deploy via the GUI: Container Manager reports failures as a generic "Failed
to create the project", while the CLI names the exact variable or line.

## Step 5 — Create the project in Container Manager

1. **Container Manager → Project → Create**
2. **Project name:** `hermes` (lowercase)
3. **Path:** _Set Path_ → `docker` → `hermes` — the project folder, not `data`
4. Choose **use the existing `docker-compose.yml`**. Do not edit it in the
   textarea; saving from that editor can reformat the file.
5. **Web portal settings:** skip. There are no published ports by design.
6. **Done.** Let the build dialog finish.

Equivalent over SSH:

```bash
cd /volume1/docker/hermes && sudo docker compose up -d
```

If you edit `docker-compose.yml` after the project exists, **Action → Build**
re-reads it from disk. If a stale error persists, **Action → Delete** and
recreate — and when prompted, **do not delete the project folder**, which holds
`.env` and `data/`.

## Step 6 — Verify

The container starting proves almost nothing. These prove the decisions.

```bash
# Healthy. PORTS must read 127.0.0.1:9119->9119/tcp — NOT 0.0.0.0.
# `health: starting` for the first 120s is expected.
sudo docker ps --filter name=hermes
sudo docker logs --tail 60 hermes

# Did the env reach the container?
sudo docker exec hermes env | grep -E 'LLM_BASE_URL|LLM_MODEL|BRAVE_MCP_URL|TZ'

# The allowlist held: expect ONLY memory, clarify, todo
sudo docker exec hermes hermes tools list
sudo docker exec hermes hermes prompt-size        # ~6.6 KB across 3 tools

# Proves the LAN inference URL works from inside the container (Finding 1)
sudo docker exec hermes hermes -z 'Reply with exactly: MINIMAL OK'

# Proves the container-to-container network join worked
sudo docker exec hermes hermes mcp test brave-search

# Dashboard answers on loopback...
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:9119/

# ...and is NOT on the LAN. This one MUST fail.
curl -s -o /dev/null -w '%{http_code}\n' --max-time 3 http://192.168.1.222:9119/
```

That last pair is the security check, not a formality. If the LAN address
answers, `BIND_ADDR` did not take and a fully capable agent is sitting on your
LAN behind nothing but a login form.

Interactive TUI (note `-it`, without which it appears broken):

```bash
sudo docker exec -it hermes hermes
```

Finally, message the bot from an allowlisted account and ask it something
deliberately under-specified. It should return a question as **tappable inline
buttons** rather than guessing — that is `clarify` working, and it confirms the
allowlist and the model path in one message.

---

## The dashboard, over the tailnet

`minimal/` ships with the dashboard off and no `ports:` key at all — Telegram
as the only interface. This deployment turns it on, published to **loopback
only** and fronted by `tailscale serve`. That is the same pattern
`brave-search-mcp` already uses on `:8422`, and it works here despite Finding 1:
`tailscale serve` is handled inside tailscaled and needs no kernel route. Only
*outbound* `100.x` from the host is broken.

**In `docker-compose.yml`:**

```yaml
      HERMES_DASHBOARD: "1"
      HERMES_DASHBOARD_HOST: "0.0.0.0"
      HERMES_DASHBOARD_PORT: "9119"
      HERMES_DASHBOARD_BASIC_AUTH_USERNAME: "${DASHBOARD_USER:?set DASHBOARD_USER in .env}"
      HERMES_DASHBOARD_BASIC_AUTH_PASSWORD: "${DASHBOARD_PASS:?set DASHBOARD_PASS in .env}"
      HERMES_DASHBOARD_BASIC_AUTH_SECRET: "${DASHBOARD_SECRET:?set DASHBOARD_SECRET in .env}"

    ports:
      - "${BIND_ADDR:-127.0.0.1}:9119:9119"
```

`HERMES_DASHBOARD_HOST` is the bind **inside** the container and must be
`0.0.0.0` for the published port to reach it. Host-side exposure is controlled
entirely by `BIND_ADDR`. Those two being different values is the part that
looks wrong and is not.

Also switch the healthcheck, now that there is an HTTP endpoint to probe:

```yaml
      test: ["CMD-SHELL", "curl -s -o /dev/null http://127.0.0.1:9119/ || exit 1"]
      start_period: 120s
```

Unauthenticated requests get a 302 to `/login`, which still proves the process
is up — so this deliberately does not use `curl -f`.

**In `.env`:**

```bash
BIND_ADDR=127.0.0.1
DASHBOARD_USER=jesse
DASHBOARD_PASS=$(openssl rand -base64 24)
DASHBOARD_SECRET=$(openssl rand -hex 32)
```

All three are mandatory — Hermes fails closed, refusing to start a non-loopback
dashboard with no auth provider. Generate them fresh for this host rather than
copying the Mac's. `DASHBOARD_SECRET` keeps sessions valid across restarts.
Despite the `BASIC_AUTH` naming it presents a `/login` form, not an HTTP auth
challenge.

**Publish it on the tailnet:**

```bash
sudo /var/packages/Tailscale/target/bin/tailscale serve --bg --https=9119 \
     http://127.0.0.1:9119

sudo /var/packages/Tailscale/target/bin/tailscale serve status
```

Reachable at **`https://ds923.tail87aa42.ts.net:9119`** from any tailnet device.

`--bg` is what makes it persistent — it is stored in tailscaled's state and
survives reboots. Without it, `serve` runs in the foreground and dies with your
shell. HTTPS certificates require MagicDNS and HTTPS to be enabled for the
tailnet, which they already are, since `:8422` works.

To undo:

```bash
sudo /var/packages/Tailscale/target/bin/tailscale serve --https=9119 off
```

> **Never set `BIND_ADDR=0.0.0.0` here.** On every NAS interface, the only
> thing between your LAN and an agent that can search the web and act on your
> behalf is that login form. Loopback plus `tailscale serve` gives you the same
> access from Mac and phone with nothing listening on the LAN.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `NanoCPUs can not be set` | DSM kernel lacks CFS bandwidth | Comment out `cpus:` — Finding 3 |
| `subsystem request failed on channel 0` | DSM SFTP subsystem off | `scp -O` — Finding 2 |
| `curl` to `100.x` times out from the NAS | Tailscale userspace mode | Use the LAN address — Finding 1 |
| `required variable X is missing a value` | A `.env` value is unset | Compose names it; fails before starting anything |
| `network brave-search-mcp_default ... could not be found` | Brave project not running | Start it first; the network is `external` |
| `TELEGRAM_BOT_TOKEN is set to a placeholder value` | Sample-looking token | Gateway starts, adapter does not |
| Telegram `409 Conflict` | Two gateways on one token | Use a separate bot per host |
| Container cannot write `/opt/data` | `PUID`/`PGID` mismatch | Re-run `id`; compare to `ls -ld` |
| MCP fails from the NAS | Using the tailnet URL | Container name + internal port |
| Dashboard container starts then dies | Non-loopback bind, no auth provider | Check `docker exec hermes env \| grep DASHBOARD` |
| Dashboard answers on `192.168.1.222:9119` | `BIND_ADDR` not applied | Must be `127.0.0.1`; recreate the container, not just restart |
| Tailnet URL dead after a reboot | `tailscale serve` started without `--bg` | Re-run with `--bg` |

## Maintenance

- **DHCP reservation for `192.168.1.50`** — the single most likely future
  breakage (Finding 1).
- **`data/config.yaml` is machine-managed.** `hermes config set` and the
  dashboard rewrite it: comments stripped, keys reordered. `${VAR}`
  placeholders survive. Treat it as seed state; keep `nas/config.yaml` on the
  Mac as the source of truth.
- **After an image update**, re-check the toolset allowlist with `hermes tools
  list` — Hermes enables toolsets it has never seen before, which is what
  `known_builtin_toolsets` in `config.yaml` exists to prevent.
- **`nas/.env` holds a bot token and the dashboard credentials.** It is
  gitignored; keep it that way, and `chmod 600` it after every `scp` — the mode
  does not survive the copy.
- **`tailscale serve` config lives in tailscaled's state**, not in this repo.
  It is not restored by redeploying the container; check it with `tailscale
  serve status` if the tailnet URL goes quiet.
