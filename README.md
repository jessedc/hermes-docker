# Hermes Agent on Synology DSM 7.2.2

This document tells you how to operate the
[Hermes Agent](https://hermes-agent.nousresearch.com/) container with Synology
**Container Manager**. A different machine on your tailnet does the inference.
That machine gives an OpenAI-compatible endpoint.

This document uses ASD-STE100 Simplified Technical English.

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

All the Hermes data is in one bind-mounted folder. The image holds no data.
Thus you make an update when you pull the image again.

## Files in this project

| File                  | Location                                      |
| --------------------- | --------------------------------------------- |
| `docker-compose.yml`  | project folder (`/volume1/docker/hermes/`)    |
| `.env.example`        | copy to `.env` in the project folder          |
| `config.yaml.example` | copy to `config.yaml` in the **data** folder  |

## Necessary conditions

Before you start, make sure that you have these items:

- DSM 7.2.2 with the **Container Manager** package (from Package Center).
- SSH on the NAS: _Control Panel → Terminal & SNMP → Enable SSH_. You use SSH
  one time for the installation. Subsequently, SSH is the most easy method to
  use the Hermes CLI.
- The **Tailscale** package on the NAS, with a login. As an alternative, use a
  subnet route to the inference host. Containers in bridge mode get access to
  `100.x.y.z` addresses through the routing table of the host. Thus you do not
  need a special network mode.
- An OpenAI-compatible server that you can get access to, and its model id.
- 4 GB of free RAM. Hermes is small. The model does not operate on the NAS.

The image `nousresearch/hermes-agent:latest` has a `linux/amd64` version and a
`linux/arm64` version. Thus it operates on the Intel models and on the ARM
models of the Synology range that can run Container Manager.

## 1. Do a test of the endpoint from the NAS

Do this test before you use Container Manager. If this test fails, all the
subsequent procedures also fail. You will then examine the incorrect layer.

Start an SSH session to the NAS and give this command:

```bash
curl -s http://100.x.y.z:8000/v1/models | head -40
```

The reply must be a JSON body with a `data` array. Make a record of the `id`
field. That value goes into `model:` in the `config.yaml` file. Do not change
the characters.

If your server needs a token, give this command:

```bash
curl -s -H "Authorization: Bearer YOUR_TOKEN" http://100.x.y.z:8000/v1/models
```

Two failures are usual:

- **Connection refused, or no route.** The inference server has a connection to
  `127.0.0.1` on its own host. Change the connection to `0.0.0.0`, or to the
  Tailscale address of that host. Then make sure that the tailnet ACLs let the
  NAS get access.
- **404.** Your base URL has an incorrect `/v1` suffix, or it has no `/v1`
  suffix. Find the path that makes `/models` operate. Point `base_url` at that
  path.

Use the `100.x.y.z` address. Do not use a MagicDNS name. Refer to
[Tailnet DNS](#tailnet-dns) for the reason.

## 2. Make the folders

Use the same SSH session:

```bash
sudo mkdir -p /volume1/docker/hermes/data
sudo chown -R "$(id -u):$(id -g)" /volume1/docker/hermes
id     # make a record of the uid= and gid= values, you need them subsequently
```

Change `/volume1` if your shared folder is on a different volume. Container
Manager makes the `docker` shared folder. If that folder does not exist, make
it in _Control Panel → Shared Folder_ first.

## 3. Copy the files to the NAS

Copy `docker-compose.yml` and `.env.example` to `/volume1/docker/hermes/`.
Copy `config.yaml.example` to `/volume1/docker/hermes/data/`. You can use File
Station, or `scp` from your Mac:

```bash
scp docker-compose.yml .env.example you@nas:/volume1/docker/hermes/
scp config.yaml.example you@nas:/volume1/docker/hermes/data/
```

Then change the names of the files on the NAS. File Station does not show
dot-files correctly. Thus use SSH:

```bash
cd /volume1/docker/hermes
mv .env.example .env
mv data/config.yaml.example data/config.yaml
chmod 600 .env
```

## 4. Fill in the `.env` file

Edit `/volume1/docker/hermes/.env` and set these values:

- `PUID` and `PGID`. Use the `uid=` and `gid=` values from step 2. **Do not
  guess these values.** The container changes to a non-root user. If that user
  is not the owner of the mount, the container cannot write to `/opt/data`.
  Then the gateway does not start.
- `DASHBOARD_USER`, `DASHBOARD_PASS` and `DASHBOARD_SECRET`. These values are
  necessary. The dashboard binds to `0.0.0.0`. Hermes does not start a
  non-loopback dashboard if it has no auth provider. Make the two secrets with
  `openssl rand -base64 24` and `openssl rand -hex 32`.
- `LLM_API_KEY`. This is the bearer token for your endpoint. If your endpoint
  has no token, write `none`.
- `HERMES_DATA`. Set this value only if your path is different from
  `/volume1/docker/hermes/data`.

The compose file refers to each of these values with `:?`. Thus a value that is
not set stops the deployment with a named error. The system does not start with
an incomplete configuration.

## 5. Point `config.yaml` at your endpoint

Edit `/volume1/docker/hermes/data/config.yaml`. Set these three values below
`model:`:

```yaml
model:
  provider: "custom"
  base_url: "http://100.x.y.z:8000/v1"
  default: "the-id-from-step-1"
  api_key: "${LLM_API_KEY}"
  context_length: 32768
```

`provider: custom` tells Hermes to use `base_url` as a general
OpenAI-compatible endpoint. Hermes then does not send the requests to a known
provider.

The key for the model is **`default`**. It is not `model`. Hermes reads
`model:` correctly, and the CLI operates with it. But `default` is the
canonical name. Hermes writes `default` to the file, and the "Main Model"
selector of the dashboard reads that name. If you write `model:`, the dashboard
shows no main model until you select one manually.

Hermes replaces `${LLM_API_KEY}` with the value from the environment of the
container. The compose file gets that value from your `.env` file. Hermes
expands `${VAR}`, but it does **not** expand `$VAR`. If a variable has no
value, Hermes keeps the text and writes a warning to the log.

Set `context_length` to the value that you gave to the inference server at
start. Many self-hosted servers do not give this value. If the Hermes estimate
is too large, you get truncation errors during a session. Correct compaction
does not occur.

If you write this file before the first start, you do not have to run `hermes
setup`. `hermes setup` is the interactive wizard that collects the API keys at
the first start.

## 6. Make the project in Container Manager

1. Open **Container Manager → Project → Create**.
2. Set **Project name** to `hermes`.
3. Set **Path** to `/volume1/docker/hermes`. This is the project folder, not
   the `data` folder.
4. Container Manager finds the `docker-compose.yml` file and asks you to use
   it. Accept.
5. Go through the web-portal wizard. Do not configure a value. You do not need
   a reverse proxy entry now.
6. Select **Done → Build**.

The first build downloads 1 GB to 2 GB. Look at the log pane. The gateway
writes its bind addresses to the log at start.

## 7. Do a test of the installation

```bash
sudo docker ps --filter name=hermes
sudo docker logs --tail 100 hermes
```

Find the dashboard on `0.0.0.0:9119`. Make sure that there are no errors about
`/opt/data` permissions. Then open this address in a browser:

```
http://<nas-ip>:9119
```

The names of the environment variables contain `BASIC_AUTH`. But the dashboard
sends you to a `/login` form. It does not send an HTTP auth challenge. Log in
with `DASHBOARD_USER` and `DASHBOARD_PASS`.

Make sure that Hermes uses your endpoint, and not a default provider:

```bash
sudo docker exec -it hermes hermes config
```

The `◆ Model` section must show your `base_url`, and your model id as
`default`.

Then do a test of the full chain with the `-z` option. This option does not
start an interactive session:

```bash
sudo docker exec hermes hermes -z "What is 17 times 23? Answer with just the number."
```

A correct answer shows that the container, the endpoint and the tool loop
operate. If you configured `auxiliary.vision`, do a test of that path also. Put
a PNG file in the data folder and ask a question about it:

```bash
sudo docker exec hermes hermes -z "Describe the colors in /opt/data/test.png"
```

## Daily operation

The dashboard is the primary interface. To use the CLI, run a command in the
container. Hermes changes to its runtime user automatically. Thus do not add
the `-u` option:

```bash
sudo docker exec -it hermes hermes chat
sudo docker exec -it hermes hermes config edit
sudo docker exec -it hermes hermes -p work gateway start   # a different profile
```

Hermes writes the log data to stdout and to the disk:

```
/volume1/docker/hermes/data/logs/gateways/default/current
```

### Hermes controls the `config.yaml` file

Hermes writes the `config.yaml` file again when you change a value on the
dashboard, or when you run `hermes config set`. The new file has no comments.
Hermes puts the keys in the canonical sequence and gives them the canonical
names. Hermes also adds a `_config_version` value. Hermes removes the keys that
the writer does not know. `model.context_length` is one of these keys. Hermes
removes it the first time that you change the main model on the dashboard.

Use the file as initial data. Do not maintain it manually. After each change on
the dashboard, examine the values that stay in the file:

```bash
docker exec hermes hermes config get model.context_length
```

Keep the `config.yaml.example` file in version control. That file has the
comments and the explanations in it.

## Optional: give access to the OpenAI-compatible API of Hermes

This API is not the same as the endpoint that Hermes uses for inference. This
API makes *Hermes* look like an OpenAI API to the other tools on your tailnet.

1. Add a strong key to the `.env` file:

   ```bash
   echo "API_SERVER_KEY=$(openssl rand -hex 32)" >> /volume1/docker/hermes/.env
   ```

2. In `docker-compose.yml`, set `API_SERVER_ENABLED: "true"`. Then make the
   `API_SERVER_HOST`, `API_SERVER_PORT` and `API_SERVER_KEY` lines active, and
   the `8642` port mapping also.
3. Build the project again in Container Manager.

Hermes needs a key of 8 characters minimum when the API server binds to
`0.0.0.0`. If the key is too short, Hermes does not start. Use this health
check:

```bash
sudo docker exec hermes curl -s http://127.0.0.1:8642/health
```

## Network

### Tailnet DNS

The Synology Tailscale package puts a `tailscale0` interface on the host. Thus
containers in bridge mode get access to `100.x.y.z` addresses, because the host
routes the data for them. **MagicDNS names are different.** The package does
not change the resolvers of the containers. Thus a name such as
`llm.your-tailnet.ts.net` usually does not resolve in the container. The same
name resolves correctly in the NAS shell.

Use the `100.x.y.z` address. If you prefer the name, set it in
`docker-compose.yml`:

```yaml
extra_hosts:
  - "llm.your-tailnet.ts.net:100.x.y.z"
```

You can also point the resolver of the container at `100.100.100.100`. But then
*all* the DNS data of the container goes through Tailscale. If Tailscale stops,
no name resolves. The `extra_hosts` entry is more reliable.

A Tailscale address does not change for a given node. Thus the `extra_hosts`
entry stays correct. If you install the inference host again, that host gets a
new address. Then change the address in `config.yaml` and in `extra_hosts`.

### Limit the access to the ports

By default, the dashboard is available on all the NAS interfaces. To make it
available only on the tailnet, put the Tailscale address of the NAS in the
`.env` file:

```bash
BIND_ADDR=100.a.b.c
```

Then build the project again. Docker binds the port to that interface only.
This procedure writes the tailnet address of the NAS into the deployment. If
that address changes, the container does not start and Docker gives a bind
error. This failure is easy to find.

The DSM firewall (_Control Panel → Security → Firewall_) is a good second
control. Use it if you want LAN access, but only from some subnets.

### Reverse proxy

For HTTPS, use _Control Panel → Login Portal → Advanced → Reverse Proxy_. It
can put `localhost:9119` behind a DSM certificate. The basic auth data goes
through without a change.

## How to update the image

Docker installations do not have the `hermes update` command. Update the image:

```bash
cd /volume1/docker/hermes
sudo docker compose pull
sudo docker compose up -d
```

As an alternative, use Container Manager. Select the project, then select
**Action → Build**. Container Manager pulls the `:latest` image again. The data
in the `data/` folder does not change. To control when the version changes,
write a specific tag in `docker-compose.yml`. Do not use `latest`.

## How to make a backup

All the important data is in one folder:

```bash
sudo tar czf /volume1/backups/hermes-$(date +%F).tar.gz \
  -C /volume1/docker/hermes data .env
```

This command includes `config.yaml`, `.env`, `SOUL.md`, the sessions, the
memories, the skills and the cron jobs. For a scheduled backup, add
`/volume1/docker/hermes` to Hyper Backup.

## Problems and solutions

**Container Manager does not accept `deploy:`.** Some DSM versions do not
accept this block. Replace the full `deploy:` block with the legacy values:

```yaml
    mem_limit: 4g
    cpus: 2.0
```

**You get permission errors on `/opt/data`, or the gateway stops immediately.**
The `PUID` and `PGID` values do not agree with the owner of the folder. Examine
the owner:

```bash
stat -c '%u %g' /volume1/docker/hermes/data
```

Then correct the `.env` file, or do the `chown` command from step 2 again.

**The dashboard container starts, then stops.** Hermes stops if the dashboard
binds to a non-loopback address and has no auth provider. Make sure that
`DASHBOARD_USER` and `DASHBOARD_PASS` are in the container:

```bash
sudo docker exec hermes env | grep DASHBOARD
```

`HERMES_DASHBOARD_INSECURE=1` is obsolete and Hermes ignores it. Thus you
cannot use it as an alternative.

**The model calls fail, but the dashboard operates.** The container cannot get
access to the endpoint. The NAS shell can get access to it. Do this test:

```bash
sudo docker exec hermes curl -s http://100.x.y.z:8000/v1/models
```

The cause is usually DNS (refer to the previous data) or a tailnet ACL.

**You get truncation errors or context errors during a session.** The
`context_length` value in `config.yaml` is larger than the value of the
inference server.

**The deployment fails with `variable is not set`.** A `:?` guard operated. The
message gives the name of the variable that is not set. Add that variable to
the `.env` file. Also make sure that the `.env` file is in the same folder as
`docker-compose.yml`. It must not be in the `data/` folder.

## Security data

**Warning: The basic auth of the dashboard is the only protection between your
LAN and an agent that can run shell commands.** Use a long random password.
Also set `BIND_ADDR=100.a.b.c`, to keep the dashboard off the LAN.

- The `.env` file holds secrets in plain text. Do a `chmod 600` on it. Do not
  let a service index or sync the `docker` shared folder.
- The Docker socket mount is not active. This is deliberate. If you mount the
  socket, the agent gets root access to the NAS and to all the other
  containers.
- The shell commands from the agent operate in this container as `PUID`. They
  can see only `/opt/data`. If you mount more host paths, the agent can see
  more data.
- Do not send port 9119 through your router. Use the tailnet.

## Appendix: operation on macOS

The compose file is Compose v2 and has nothing that is specific to Synology.
The image has a `linux/arm64` version. Thus the same stack operates with Docker
Desktop. Only the values in the `.env` file are different. Steps 2, 3 and 6 are
Container Manager procedures. On macOS, use `docker compose up -d` in their
place.

```bash
HERMES_DATA=/Users/you/Development/hermes-docker/data
PUID=501            # your uid; it has almost no function with Docker Desktop,
PGID=20             # which changes bind-mount ownership through virtiofs
BIND_ADDR=127.0.0.1 # dashboard on loopback only
MEM_LIMIT=2G        # must be less than the memory of the Docker Desktop VM
```

`MEM_LIMIT` causes the most problems. Compose accepts a limit that is larger
than the VM. Then the container gets an OOM-kill during operation. It does not
fail at start.

### How to use the DGX Spark on the tailnet

This is the configuration that is in use. Hermes operates with Docker Desktop
on the Mac. The inference is on a DGX Spark (`edgexpert-7b1e`, NVIDIA GB10).
The Spark runs llama.cpp on port 8080. The Mac gets access to it on the
tailnet.

Docker Desktop sends container data for `100.x.y.z` addresses through the host.
Thus you do not need `host.docker.internal` and you do not need a special
network mode. Do a test from a container before you configure Hermes. A correct
`curl` on the Mac does not show that the container has access:

```bash
docker run --rm curlimages/curl -s -o /dev/null -w '%{http_code}\n' \
  http://100.x.y.z:8080/v1/models          # the reply must be 200
```

Then get the data about the server. `/v1/models` gives the id and the context
window from the start command. `/props` gives the modalities:

```bash
curl -s http://100.x.y.z:8080/v1/models | jq '.data[] | {id, n_ctx: .meta.n_ctx}'
curl -s http://100.x.y.z:8080/props    | jq '.modalities'
```

A multimodal GGUF file gives this reply:
`{"vision": true, "video": true, "audio": false}`. Thus one model does the text
and the images:

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

Write all the values in the vision block. Do not use `provider: auto`. `auto`
uses the main model, which is the correct endpoint here. But `auto` also needs
Hermes to know that the model has a vision capability. A local GGUF file has no
capability metadata for Hermes to read. If you write the values, Hermes does
not make an estimate. The other `auxiliary` items can stay on `auto`. They use
the main model.

llama.cpp examines the bearer token only when you start it with `--api-key`.
Thus `LLM_API_KEY=none` has no function here. But keep a value in that
variable. An empty key can cause a client-side validation error before the
client sends the request.

Take the `context_length` value from the `n_ctx` value of the server. Do not
use the maximum value of the model architecture. Hermes does a compaction at
50% of the `context_length` value. If that value is too large, the sessions
become very large before the compaction.

Do the tests through the container. Do not do them against the server:

```bash
docker exec hermes hermes -z 'Reply with exactly: SPARK OK'
docker exec hermes hermes -z 'Use your vision capability on /opt/data/workspace/test.png. Reply with only the shape and its color.'
```

Use an image for which you know the correct answer. A flat colored shape on a
white background is sufficient. It shows the difference between a correct
vision path and a hallucination.

One operational note: llama.cpp gives `total_slots: 1` by default. Thus the
requests go into a queue and do not operate in parallel. A vision request
during a long generation waits. This looks like a failure if you do not know
about the queue. Examine the current task of the server:

```bash
curl -s http://100.x.y.z:8080/slots | jq '.[0] | {is_processing, n_prompt_tokens}'
```

## Reference

- [Hermes Docker guide](https://hermes-agent.nousresearch.com/docs/user-guide/docker/)
- [Configuration reference](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)
- [Security](https://hermes-agent.nousresearch.com/docs/user-guide/security)
