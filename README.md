# Hermes Agent on Synology DSM 7.2.2

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
Replace the whole `deploy:` block with the legacy equivalents:

```yaml
    mem_limit: 4g
    cpus: 2.0
```

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

A multimodal GGUF answers `{"vision": true, "video": true, "audio": false}`,
which means one model covers both text and images:

```yaml
model:
  provider: "custom"
  base_url: "http://100.x.y.z:8080/v1"
  default: "unsloth/Muse-Glimmer-30B-GGUF:UD-Q6_K_XL"
  api_key: "${LLM_API_KEY}"
  context_length: 131072

auxiliary:
  vision:
    provider: "custom"
    model: "unsloth/Muse-Glimmer-30B-GGUF:UD-Q6_K_XL"
    base_url: "http://100.x.y.z:8080/v1"
    api_key: "${LLM_API_KEY}"
    timeout: 180
```

Spell the vision block out rather than leaving it on `provider: auto`. `auto`
reuses the main model, which is the right endpoint here, but it depends on
Hermes deciding that model is vision-capable — and a locally-served GGUF carries
no capability metadata for it to consult. Being explicit removes the guess. The
other `auxiliary` slots can stay on `auto`; they follow the main model.

llama.cpp only checks the bearer token when started with `--api-key`, so
`LLM_API_KEY=none` is an ignored placeholder here. Keep it non-empty regardless
— an empty key can trip client-side validation before a request is even sent.

Take `context_length` from the server's own `n_ctx`, not the model's
architectural maximum. Hermes compacts at 50% of this value, so an inflated
number means sessions balloon before anything trims them.

Verify both paths through the container rather than against the server:

```bash
docker exec hermes hermes -z 'Reply with exactly: SPARK OK'
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

## Reference

- [Hermes Docker guide](https://hermes-agent.nousresearch.com/docs/user-guide/docker/)
- [Configuration reference](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)
- [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security)
