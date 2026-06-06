---
name: install-spur-cloud
description: Use when the user asks to install/deploy Spur Cloud (the GPUaaS web platform) on a single bare-metal host via SSH. Stands up PostgreSQL + spurctld + spurd + spur-cloud-api + frontend on ONE node using the `bare_metal` backend (no K8s). Always asks the user up-front for target host, Spur Cloud version, and auth method before touching anything.
---

# Install Spur Cloud (single bare-metal node)

Spur Cloud is the GPUaaS web platform built on Spur (the scheduler). The minimum viable deployment is:

| Component | Where | Why |
|---|---|---|
| **PostgreSQL 16** | Docker on the host | Platform DB (users, sessions, SSH keys, billing) |
| **spurctld** | Native binary | Spur controller (gRPC :6817, Raft :6821) |
| **spurd** | Native binary | Spur agent (:6818) — runs jobs as bare processes |
| **spur-cloud-api** | Docker | Rust/axum backend (:8080) |
| **spur-cloud-frontend** | Docker (nginx) | React SPA (:80) — proxies `/api` to the API container |

Single-node uses `backend = "bare_metal"` in `spur-cloud.toml` — this tells the API to skip K8s entirely and run sessions as native processes via spurd. The API server's docs (`spur-cloud.toml.example`, `README.md`, `crates/spur-cloud-api/src/config.rs::BareMetalConfig`) are the source of truth; this skill follows them.

> **Naming note:** Earlier drafts of this skill called the backend `native_host`. The shipped enum (v0.3.0+) is `bare_metal` — both the `server.backend` value and the section header `[bare_metal]`. If you see `native_host` anywhere, it's stale.

Defaults (override only if user asks):

| Var | Default |
|---|---|
| `SC_HOME` | `/root/spur-cloud` |
| `SC_INSTALL_DIR` | `/root/.local/bin` |
| `SC_VERSION` | `latest` (`vX.Y.Z` for pinned; resolves the matching git tag) |
| `SC_SPUR_VERSION` | `latest` (passed to spur `install.sh`) |
| `SC_API_PORT` | `8080` |
| `SC_FRONTEND_PORT` | `80` |
| `SC_PG_PORT` | `5432` |
| `SC_PG_PASSWORD` | random 32-hex (`openssl rand -hex 16`) |
| `SC_JWT_SECRET` | random 64-hex (`openssl rand -hex 32`) |
| `SC_SSH_PORT_BASE` | `10000` |
| `SC_SSH_PORT_RANGE` | `1000` (session SSH NodePort range) |
| `SC_PUBLIC_URL` | `http://<host>:${SC_FRONTEND_PORT}` |
| `SC_WIPE` | `false` (set `true` to nuke existing DB + Raft on redeploy) |
| SSH user | `root` (unless the user says otherwise) |

## Step 0: gather inputs (MANDATORY — do not skip)

Before any SSH, **ask the user** for:

1. **Target host** — single SSH target (e.g. `root@gpu01.example.com`). The skill installs on ONE host only. For multi-node Spur Cloud, the user should use Helm/K8s instead.
2. **Spur Cloud version** — `latest` (default), a `vX.Y.Z` git tag, or `main` (HEAD of upstream). The skill builds the API + frontend Docker images locally from this ref.
3. **Auth method** — pick one:
   | Mode | What it does |
   |---|---|
   | `local` | Email/password only (Argon2). Simplest. **Default.** |
   | `github` | GitHub OAuth2 — needs `client_id` + `client_secret` from a GitHub OAuth App. |
   | `okta` | Okta OIDC — needs `issuer`, `client_id`, `client_secret`, and optional `admin_groups`. |
4. **GPU vendor** — `amd` / `nvidia` / `none`. Only affects what runtime hooks spurd installs and what env var (`ROCR_VISIBLE_DEVICES` / `CUDA_VISIBLE_DEVICES`) sessions see. Pick `none` for a CPU-only smoke test.

Use `AskUserQuestion` for any of these the user didn't specify. Don't guess. If the user picks `github` or `okta`, prompt for the credentials explicitly — those are required, not optional.

> **Warn the user** that this skill is for single-BM ONLY:
> - No HA (1 spurctld, 1 spurd, 1 Postgres container).
> - SSH-to-session works (port-forward via spurd), but no K8s NodePort.
> - For multi-node or HA, point them at `deploy/k8s/` (Kustomize) or the Helm chart on the `helm-chart` branch.

Once gathered, **write env vars to a persistent file and source it for every subsequent step.** Do NOT regenerate secrets inline in later steps — the random defaults (`openssl rand …`) produce a new value on every evaluation, and a fresh `source` between Step 5 (PG container start) and Step 7 (TOML render) will leave the API authenticating with a different password than PG was started with. The symptom is a restart loop with `password authentication failed for user "spur_cloud"`.

Generate secrets **exactly once** here, write them to `/tmp/sc-env.sh` on the control host, and `source /tmp/sc-env.sh` at the top of every later step's Bash block:

```bash
cat > /tmp/sc-env.sh <<EOF
export TGT=root@gpu01.example.com
export ADDR=\${TGT#*@}
export SC_VERSION=v0.3.0          # or 'latest' / 'main'
export AUTH_MODE=local            # or 'github' / 'okta'
export GPU_VENDOR=amd             # or 'nvidia' / 'none'
export SC_HOME=/root/spur-cloud
export SC_INSTALL_DIR=/root/.local/bin
export SC_SPUR_VERSION=latest
export SC_API_PORT=8080
export SC_FRONTEND_PORT=80
export SC_PG_PORT=5432
export SC_SSH_PORT_BASE=10000
export SC_SSH_PORT_RANGE=1000
# Secrets — generated ONCE here; do NOT re-roll later.
export SC_PG_PASSWORD=$(openssl rand -hex 16)
export SC_JWT_SECRET=$(openssl rand -hex 32)
export SC_PUBLIC_URL="http://\${ADDR}:\${SC_FRONTEND_PORT}"
EOF
chmod 600 /tmp/sc-env.sh   # plaintext secrets — restrict to owner. Delete or relocate after deploy.
source /tmp/sc-env.sh
```

> **Secrets hygiene:** `/tmp/sc-env.sh` holds PG and JWT secrets in plaintext on the control host. `chmod 600` it now, and delete or move it to a secret store after the deploy completes (the same values are persisted on the target in `${SC_HOME}/etc/spur-cloud.toml` and the PG container env).

## Step 1: preflight the host

Fail-fast if any of these check fails — do NOT try to "fix" the host silently.

```bash
ssh -o BatchMode=yes -o ConnectTimeout=10 "$TGT" '
  set +e
  echo "host=$(hostname -s) fqdn=$(hostname -f)"
  echo "kernel=$(uname -r) nproc=$(nproc)"

  echo "--- spur-cloud ports (80/8080/5432, 6817/6818/6821) ---"
  ss -tlnpH 2>/dev/null | grep -E ":(80|8080|5432|6817|6818|6821)\b" || echo "ports free"

  echo "--- existing spur-cloud processes ---"
  pgrep -ax spurctld; pgrep -ax spurd; echo "(end pids)"

  echo "--- docker ---"
  if ! command -v docker >/dev/null; then echo "MISSING:docker"; fi
  docker info >/dev/null 2>&1 || echo "DOCKER_NOT_USABLE (need root or docker group)"
  docker ps --format "{{.Names}}" 2>/dev/null | grep -E "^(spur-cloud-|spur-cloud-pg)" || echo "no spur-cloud containers"

  echo "--- tools ---"
  for t in curl tar bash ss pgrep pkill nohup git openssl; do
    command -v $t >/dev/null || echo "MISSING:$t"
  done

  echo "--- gpu ---"
  if [ -e /dev/kfd ]; then echo "amd-gpu-detected (rocm)"; fi
  command -v nvidia-smi >/dev/null && echo "nvidia-gpu-detected"

  echo "--- os ---"
  . /etc/os-release 2>/dev/null && echo "$PRETTY_NAME"

  echo "--- disk ---"
  df -h /var/lib/docker /root 2>/dev/null
'
```

Abort conditions:
- `MISSING:docker` or `DOCKER_NOT_USABLE` → ask user to install Docker first.
- `MISSING:curl|tar|git|openssl` → ask user to install them.
- A spur-cloud port (`80/8080/5432`) is held by a non-spur-cloud process → abort.
- GPU vendor the user picked doesn't match what's on the host → warn, then ask whether to continue.

(Existing spur / spur-cloud containers and daemons are fine — we stop and wipe them in Step 4.)

## Step 2: install Spur (spurctld + spurd)

Spur Cloud needs Spur underneath. We use Spur's upstream installer — it drops `spur`, `spurctld`, `spurd`, etc. into `${SC_INSTALL_DIR}`. **Do NOT** depend on the spur repo's `deploy-spur` skill; this skill stands alone.

```bash
ssh "$TGT" "
  set -euo pipefail
  mkdir -p ${SC_HOME} ${SC_HOME}/state ${SC_HOME}/log ${SC_HOME}/etc ${SC_HOME}/work ${SC_INSTALL_DIR}
  if ! ${SC_INSTALL_DIR}/spur --version >/dev/null 2>&1; then
    curl -fsSL https://raw.githubusercontent.com/ROCm/spur/main/install.sh \
      | INSTALL_DIR=${SC_INSTALL_DIR} bash -s -- ${SC_SPUR_VERSION}
  fi
  ${SC_INSTALL_DIR}/spur --version
"
```

## Step 3: clone spur-cloud source on the host and build images

We build the API + frontend Docker images on the target host so the skill works for `latest`, any tag, or `main` without depending on a published image registry.

```bash
ssh "$TGT" "
  set -euo pipefail
  cd ${SC_HOME}
  if [ ! -d src/.git ]; then
    git clone https://github.com/ROCm/spur-cloud.git src
  fi
  cd src
  git fetch --tags origin
  if [ '${SC_VERSION}' = 'latest' ]; then
    # Resolve 'latest' to the newest semver tag (vX.Y.Z), fall back to main if none.
    REF=\$(git tag -l 'v*' | sort -V | tail -1)
    [ -z \"\$REF\" ] && REF=origin/main
  elif [ '${SC_VERSION}' = 'main' ]; then
    REF=origin/main
  else
    REF='${SC_VERSION}'
  fi
  echo \"checking out \$REF\"
  git checkout --quiet \"\$REF\"
"
```

Then build the images. **Important:** the API Dockerfile expects the build context at the repo root (it copies `Cargo.toml`/`crates/`); the frontend Dockerfile copies `frontend/`. Both run from the repo root with the right `-f` flag.

```bash
ssh "$TGT" "
  set -euo pipefail
  cd ${SC_HOME}/src
  docker build -f deploy/docker/Dockerfile.api      -t spur-cloud-api:local      .
  docker build -f deploy/docker/Dockerfile.frontend -t spur-cloud-frontend:local .
"
```

The API build runs `cargo build --release` inside the container — first build takes ~10-20 min depending on CPU. Subsequent builds reuse Docker layer cache.

## Step 4: stop any existing spur-cloud + spur processes (idempotent)

Order: containers first, then native daemons. Then wipe state ONLY if user asked (`SC_WIPE=true`).

```bash
ssh "$TGT" "
  docker rm -f spur-cloud-frontend spur-cloud-api spur-cloud-pg 2>/dev/null || true
  # pkill -x (exact name) — never -f, that also kills spurctld.
  pkill -x spurd 2>/dev/null || true
  pkill -x spurctld 2>/dev/null || true
  for i in \$(seq 1 10); do
    pgrep -x spurctld >/dev/null || pgrep -x spurd >/dev/null || exit 0
    sleep 0.5
  done
"

# Wipe (only when explicitly requested)
if [ "${SC_WIPE:-false}" = true ]; then
  ssh "$TGT" "
    rm -rf ${SC_HOME}/state ${SC_HOME}/pg-data
    mkdir -p ${SC_HOME}/state ${SC_HOME}/pg-data
  "
fi
```

## Step 5: start PostgreSQL

```bash
ssh "$TGT" "
  mkdir -p ${SC_HOME}/pg-data
  docker run -d --name spur-cloud-pg \
    --restart unless-stopped \
    -e POSTGRES_DB=spur_cloud \
    -e POSTGRES_USER=spur_cloud \
    -e POSTGRES_PASSWORD='${SC_PG_PASSWORD}' \
    -p 127.0.0.1:${SC_PG_PORT}:5432 \
    -p 172.17.0.1:${SC_PG_PORT}:5432 \
    -v ${SC_HOME}/pg-data:/var/lib/postgresql/data \
    postgres:16

  # Wait for Postgres to accept connections.
  for i in \$(seq 1 30); do
    docker exec spur-cloud-pg pg_isready -U spur_cloud >/dev/null 2>&1 && { echo 'pg ready'; exit 0; }
    sleep 1
  done
  echo 'PG never became ready' >&2; docker logs --tail=40 spur-cloud-pg; exit 1
"
```

PG is published on two interfaces:
- `127.0.0.1:5432` — for host-local tooling (psql, etc.)
- `172.17.0.1:5432` — the default Linux `docker0` bridge gateway. The API container reaches PG via `host.docker.internal`, which on Linux resolves to the docker0 gateway (172.17.0.1). Without this second publish, the API gets `pool timed out` because `127.0.0.1:5432` only accepts loopback traffic, not the bridge gateway.

If `docker network inspect bridge` shows the gateway is something other than `172.17.0.1` (custom Docker config), substitute that IP. A cleaner long-term layout is a user-defined Docker network with PG + API attached so PG is reachable by service name — that removes the `host.docker.internal` dance entirely.

## Step 6: render spur.conf and start Spur (single-node hyperconverged)

This is a 1-controller + 1-agent on the same host. `node_id`/`peers` are still required because spurctld always runs as a 1-member Raft cluster.

```bash
SHORT=$(ssh "$TGT" 'hostname -s')
ADDR=${TGT#*@}                  # SSH target hostname/IP
CPUS=$(ssh "$TGT" 'nproc')
MEM_MB=$(ssh "$TGT" "awk '/MemTotal/{print int(\$2/1024*9/10)}' /proc/meminfo")

ssh "$TGT" "cat > ${SC_HOME}/etc/spur.conf" <<EOF
cluster_name = "spur-cloud"

[controller]
listen_addr = "[::]:6817"
hosts = ["${ADDR}"]
state_dir = "${SC_HOME}/state"
raft_listen_addr = "[::]:6821"
node_id = 1
peers = ["${ADDR}:6821"]

[scheduler]
plugin = "backfill"
interval_secs = 1

[network]
wg_enabled = false
agent_port = 6818

[[nodes]]
names = "${SHORT}"
cpus = ${CPUS}
memory_mb = ${MEM_MB}

[[partitions]]
name = "default"
default = true
nodes = "${SHORT}"
max_time = "INFINITE"
EOF
```

Start spurctld and spurd. Gotchas (all hard-won — see "Gotchas" at the bottom):
- `-D` means **FOREGROUND** in Spur; do NOT pass it when backgrounding.
- `nohup … < /dev/null & disown` — both mandatory.
- `cd ${SC_HOME}/work` before launching spurd so job stdout (`spur-<N>.out`) lands somewhere predictable.

```bash
ssh "$TGT" "
  nohup ${SC_INSTALL_DIR}/spurctld \
      -f ${SC_HOME}/etc/spur.conf \
      --state-dir ${SC_HOME}/state \
      --log-level info \
      > ${SC_HOME}/log/spurctld.log 2>&1 < /dev/null &
  disown
  for i in \$(seq 1 30); do
    ss -tlnH | grep -q ':6817\b' && { echo 'spurctld up'; break; }
    sleep 1
  done
  # Wait for Raft leader (single-member quorum forms in ~1s).
  for i in \$(seq 1 30); do
    out=\$(${SC_INSTALL_DIR}/spur nodes 2>&1)
    echo \"\$out\" | grep -qE 'no leader|not the Raft leader|Connection refused' || break
    sleep 1
  done

  cd ${SC_HOME}/work
  nohup ${SC_INSTALL_DIR}/spurd \
      --controller http://127.0.0.1:6817 \
      --hostname ${SHORT} \
      --address ${ADDR} \
      --listen 0.0.0.0:6818 \
      --log-level info \
      > ${SC_HOME}/log/spurd.log 2>&1 < /dev/null &
  disown
  for i in \$(seq 1 30); do
    ss -tlnH | grep -q ':6818\b' && { echo 'spurd up'; exit 0; }
    sleep 1
  done
  echo 'spurd never bound 6818' >&2; exit 1
"

# Confirm registration
ssh "$TGT" "for i in \$(seq 1 30); do ${SC_INSTALL_DIR}/spur show node ${SHORT} >/dev/null 2>&1 && { echo registered; exit 0; }; sleep 1; done; echo NEVER_REGISTERED >&2; exit 1"
```

## Step 7: render spur-cloud.toml

The config lives on the host (mounted into the API container). For `auth.local`, no extra section is needed. For `github`/`okta`, append the credentials the user gave in Step 0.

```bash
ssh "$TGT" "cat > ${SC_HOME}/etc/spur-cloud.toml" <<EOF
public_url = "${SC_PUBLIC_URL}"

[server]
listen_addr = "0.0.0.0:8080"
session_namespace = "spur-sessions"
backend = "bare_metal"

[bare_metal]
agent_port = 6818
ssh_port_base = ${SC_SSH_PORT_BASE}
ssh_port_range = ${SC_SSH_PORT_RANGE}

[database]
url = "postgresql://spur_cloud:${SC_PG_PASSWORD}@host.docker.internal:${SC_PG_PORT}/spur_cloud"

[spur]
controller_addr = "http://host.docker.internal:6817"

[auth]
jwt_secret = "${SC_JWT_SECRET}"
jwt_expiry_hours = 24
EOF
```

If `AUTH_MODE=github`:

```bash
ssh "$TGT" "cat >> ${SC_HOME}/etc/spur-cloud.toml" <<EOF

[auth.github]
enabled = true
client_id = "${GITHUB_CLIENT_ID}"
client_secret = "${GITHUB_CLIENT_SECRET}"
EOF
```

If `AUTH_MODE=okta`:

```bash
ssh "$TGT" "cat >> ${SC_HOME}/etc/spur-cloud.toml" <<EOF

[auth.okta]
enabled = true
issuer = "${OKTA_ISSUER}"
client_id = "${OKTA_CLIENT_ID}"
client_secret = "${OKTA_CLIENT_SECRET}"
admin_groups = [${OKTA_ADMIN_GROUPS_TOML}]   # e.g. "gpu-admins", "platform-eng"
EOF
```

Note: `host.docker.internal` works on Docker for Linux **only** if the container is started with `--add-host=host.docker.internal:host-gateway`. We do that in Step 8. If the user is running an old Docker (< 20.10), substitute the host IP from `ip -4 addr show docker0` instead.

## Step 8: start spur-cloud-api

```bash
ssh "$TGT" "
  docker run -d --name spur-cloud-api \
    --restart unless-stopped \
    --add-host=host.docker.internal:host-gateway \
    -p 127.0.0.1:${SC_API_PORT}:8080 \
    -p 172.17.0.1:${SC_API_PORT}:8080 \
    -v ${SC_HOME}/etc/spur-cloud.toml:/etc/spur-cloud.toml:ro \
    spur-cloud-api:local \
    --config /etc/spur-cloud.toml
    # NOTE: do NOT pass `spur-cloud-api` as the first CMD arg — the Dockerfile
    # already sets ENTRYPOINT ["spur-cloud-api"], so clap sees the extra binary
    # name as a positional arg and rejects it with `unexpected argument`.

  # Wait for /api/health or just the listen port.
  for i in \$(seq 1 60); do
    curl -fsS http://127.0.0.1:${SC_API_PORT}/api/auth/providers >/dev/null 2>&1 && { echo 'api up'; exit 0; }
    sleep 1
  done
  echo 'API never came up' >&2; docker logs --tail=80 spur-cloud-api; exit 1
"
```

Two publishes, mirroring the PG pattern from Step 5:
- `127.0.0.1:${SC_API_PORT}:8080` — for host-local debugging (`curl http://127.0.0.1:8080/...`).
- `172.17.0.1:${SC_API_PORT}:8080` — the docker0 bridge gateway. The frontend container's nginx resolves `host.docker.internal` to this IP; without the second publish, `proxy_pass http://host.docker.internal:8080` returns **502 Bad Gateway** even though host-local curl works. Substitute the gateway from `docker network inspect bridge` if your Docker uses a non-default subnet.

If you want the API reachable from outside the host for debugging, add `-p 0.0.0.0:${SC_API_PORT}:8080`.

## Step 9: start spur-cloud-frontend

The frontend Dockerfile bakes `proxy_pass http://spur-cloud-api:8080;` into its nginx config — intended for K8s/Compose where `spur-cloud-api` is a service hostname. On a bare bridge network that hostname doesn't resolve and **nginx crashes on startup** (`host not found in upstream`), so `docker exec ... nginx -s reload` never gets a chance — the container is in a restart loop. We mount a pre-fixed config at the bind mount instead, so the container starts cleanly the first time.

```bash
ssh "$TGT" "
  cat > ${SC_HOME}/etc/nginx-default.conf <<'NGINX'
server {
    listen 80;
    root /usr/share/nginx/html;
    location / {
        try_files \\\$uri \\\$uri/ /index.html;
    }
    location /api {
        proxy_pass http://host.docker.internal:${SC_API_PORT};
        proxy_http_version 1.1;
        proxy_set_header Upgrade \\\$http_upgrade;
        proxy_set_header Connection \"upgrade\";
        proxy_set_header Host \\\$host;
    }
}
NGINX

  docker run -d --name spur-cloud-frontend \
    --restart unless-stopped \
    --add-host=host.docker.internal:host-gateway \
    -p ${SC_FRONTEND_PORT}:80 \
    -v ${SC_HOME}/etc/nginx-default.conf:/etc/nginx/conf.d/default.conf:ro \
    spur-cloud-frontend:local

  for i in \$(seq 1 30); do
    curl -fsS http://127.0.0.1:${SC_FRONTEND_PORT}/ >/dev/null && { echo 'frontend up'; exit 0; }
    sleep 1
  done
  echo 'frontend never came up' >&2; docker logs --tail=40 spur-cloud-frontend; exit 1
"
```

Note the heredoc uses `'NGINX'` (single-quoted delimiter) so the shell doesn't expand `\$uri`/`\$http_upgrade`/etc. on the remote side. The triple-escaped `\\\$` (here-doc inside `ssh "..."`) survives both the local and remote shells and lands as literal `$` in the file. If you change quoting, re-verify the rendered file with `cat` before starting the container.

A cleaner long-term layout is a user-defined Docker network with all three containers attached so `proxy_pass http://spur-cloud-api:8080;` resolves natively — but that also requires moving Postgres onto the network. The mounted config above keeps the skill simple.

## Step 10: smoke test

### A. Health endpoints

```bash
ssh "$TGT" "
  echo '--- providers ---'
  curl -fsS http://127.0.0.1:${SC_API_PORT}/api/auth/providers
  echo
  echo '--- frontend root ---'
  curl -fsS http://127.0.0.1:${SC_FRONTEND_PORT}/ | head -3
  echo
  echo '--- spur nodes ---'
  ${SC_INSTALL_DIR}/spur nodes
"
```

### B. Submit a job through Spur directly (proves the scheduler half works)

```bash
ssh "$TGT" "cat > /tmp/sc-smoke.sh <<'EOF'
#!/bin/bash
#SBATCH --job-name=sc-smoke
echo \"hello from \$(hostname) at \$(date)\"
EOF
chmod +x /tmp/sc-smoke.sh
cd ${SC_HOME}/work
jid=\$(${SC_INSTALL_DIR}/spur submit /tmp/sc-smoke.sh | grep -oE '[0-9]+')
echo JOBID=\$jid
for i in \$(seq 1 30); do
  st=\$(${SC_INSTALL_DIR}/spur show job \$jid 2>/dev/null | grep -oE 'JobState=[A-Z]+' | head -1 | cut -d= -f2)
  case \"\$st\" in COMPLETED) echo OK; break ;; FAILED|CANCELLED|TIMEOUT|NODE_FAIL) echo BAD:\$st >&2; exit 1 ;; esac
  sleep 1
done
ls ${SC_HOME}/work/spur-\${jid}.out 2>/dev/null && cat ${SC_HOME}/work/spur-\${jid}.out
"
```

### C. Register a local user via the API (proves the platform half works)

Only run this if `AUTH_MODE=local`. With OAuth, the user has to go through the browser flow.

```bash
ssh "$TGT" "
  curl -fsS -X POST http://127.0.0.1:${SC_API_PORT}/api/auth/register \
    -H 'Content-Type: application/json' \
    -d '{\"email\":\"admin@example.com\",\"username\":\"admin\",\"password\":\"smoke-test-pw\"}'
  # NOTE: RegisterRequest (crates/spur-cloud-api/src/routes/auth.rs) takes
  # {email, username, password}. There is no `display_name` field — sending
  # one returns 422 Unprocessable Entity.
  echo
  echo '--- login ---'
  curl -fsS -X POST http://127.0.0.1:${SC_API_PORT}/api/auth/login \
    -H 'Content-Type: application/json' \
    -d '{\"email\":\"admin@example.com\",\"password\":\"smoke-test-pw\"}'
"
```

A successful login returns a JWT. The user can now visit `${SC_PUBLIC_URL}` in a browser.

## Step 11: verify + report

```bash
ssh "$TGT" "
  echo '=== containers ==='
  docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' | grep spur-cloud || echo '(none)'
  echo '=== native daemons ==='
  pgrep -ax spurctld; pgrep -ax spurd
  echo '=== logs ==='
  for f in spurctld.log spurd.log; do
    echo \"--- ${SC_HOME}/log/\$f (last 5 lines) ---\"
    tail -5 ${SC_HOME}/log/\$f
  done
  echo '=== nodes ==='
  ${SC_INSTALL_DIR}/spur nodes
"
```

Report back with:
- Target host + version (resolved git ref)
- Auth mode + (for `local`) the smoke-test user credentials, OR for OAuth the URL the user must visit
- Component status table (postgres / spurctld / spurd / api / frontend → up?)
- `${SC_PUBLIC_URL}` so the user knows where to point a browser
- Log paths and how to follow them (`docker logs -f spur-cloud-api`, `tail -F ${SC_HOME}/log/spurctld.log`)
- Where job stdout lands (`${SC_HOME}/work/spur-<JOBID>.out`)

## Step 12: teardown (only when explicitly asked)

```bash
ssh "$TGT" "
  docker rm -f spur-cloud-frontend spur-cloud-api spur-cloud-pg 2>/dev/null || true
  pkill -x spurd 2>/dev/null || true
  pkill -x spurctld 2>/dev/null || true
  # Wipe state + pg data + cloned source. Skip if user wants to redeploy with state preserved.
  rm -rf ${SC_HOME}
  # Sweep stray job stdout (spurd writes spur-<N>.out into its CWD at startup;
  # the CWD may have drifted across redeploys).
  rm -f /root/spur-*.out ${SC_INSTALL_DIR}/spur-*.out /tmp/spur-*.out
"
```

If the user wants to keep their data: `docker stop` the containers instead of `rm -f`, and skip the `rm -rf ${SC_HOME}`.

## Gotchas (every one of these has bitten me before)

### Spur process management
- **`-D` means FOREGROUND**, not daemonize. Use `nohup ... < /dev/null & disown`; never pass `-D` when backgrounding.
- **`pkill -f spurd` also kills `spurctld`** (substring match). Always `pkill -x`.
- **SSH backgrounding hangs without both `< /dev/null` and `disown`.**
- **`mkdir -p {a,b,c}` brace expansion can fail under `set -e` over SSH** — write explicit paths.

### Spur runtime
- **Job stdout `spur-<N>.out` lands in spurd's CWD at startup**, not the submitter's CWD. `cd ${SC_HOME}/work` before launching spurd so it's predictable.
- **`spur nodes` collapses by partition**. To verify per-host registration, loop `spur show node <name>`.
- **`spur show node <name>` does prefix match**, not exact. If two node names share a prefix you get both.
- **`spur show job` uses `JobState=COMPLETED` (uppercase)** — parse with `grep -oE 'JobState=[A-Z]+'`, not `State: Completed`.
- **Raft port 6821 is hardcoded** in spurctld — preflight must include it.
- **spurctld binds 6817 immediately but returns `no leader elected yet`** for ~1s after start until its single-member Raft quorum forms. Loop on the error string in `spur nodes`, don't just wait for port-bind.

### Spur Cloud specifics
- **Backend MUST be `bare_metal`** for single-BM (both the `server.backend` value and the section header `[bare_metal]`). Otherwise the API tries to talk to a K8s API server on startup and crashes. The enum is in `crates/spur-cloud-api/src/config.rs::Backend` — only `k8s` and `bare_metal` are valid; anything else (including the older `native_host` naming) is rejected at TOML parse time.
- **`host.docker.internal` only works on Linux Docker ≥ 20.10** AND only when the container was started with `--add-host=host.docker.internal:host-gateway`. Both are baked into the run commands above.
- **`spur-cloud-api` binds 8080** inside the container regardless of what `listen_addr` says in spur-cloud.toml — `listen_addr` *is* the bind, but the container's published port is 8080 by Dockerfile convention. Map `${SC_API_PORT}:8080`.
- **`session_namespace`** is required in the TOML even for `bare_metal` (defaulted to `"spur-sessions"`); it's ignored in bare-metal mode but the parser still wants it.
- **API container ENTRYPOINT is `spur-cloud-api`** — pass ONLY flags as CMD (e.g. `--config /etc/spur-cloud.toml`). Do not prepend the binary name a second time; clap rejects `unexpected argument 'spur-cloud-api' found`.
- **`host.docker.internal` requires both `--add-host=...:host-gateway` AND a PG publish on the bridge gateway IP.** Docker doesn't auto-route `127.0.0.1:5432` (loopback) traffic from a bridged container — `host.docker.internal` resolves to the docker0 gateway (172.17.0.1 by default), and PG must be listening there. Without the second `-p 172.17.0.1:5432:5432`, the API spins forever on `pool timed out while waiting for an open connection`.
- **Same trap for the API container, reached from the frontend.** The frontend nginx proxies `/api` to `http://host.docker.internal:8080`, which also resolves to 172.17.0.1. If the API is only published on `127.0.0.1:8080`, browser login returns `502 Bad Gateway` from nginx even though host-local `curl http://127.0.0.1:8080/...` works. Step 8 publishes on both 127.0.0.1 AND 172.17.0.1 for this reason.
- **Frontend nginx config hardcodes `proxy_pass http://spur-cloud-api:8080;`** in the upstream Dockerfile (intended for K8s/Compose). On a plain Docker bridge that hostname doesn't resolve and nginx crashes on startup with `host not found in upstream` — never reaching `nginx -s reload`. Mount a pre-fixed `default.conf` at run time instead of trying to `docker exec sed ... && nginx -s reload`.
- **Local-auth register payload is `{email, username, password}`** — there is no `display_name` field. The shipped `RegisterRequest` (`crates/spur-cloud-api/src/routes/auth.rs`) returns 422 if you send anything else.
- **OAuth callback URL**: GitHub/Okta apps must register `${SC_PUBLIC_URL}/api/auth/github/callback` (or `.../okta/callback`) as an allowed redirect. The skill prints `SC_PUBLIC_URL` at the end — the user is responsible for configuring the OAuth app.
- **First API container build takes 10-20 minutes** because `cargo build --release` runs inside the build. Don't add a per-step timeout that would kill this — the build runs on the host, not in the skill harness.
- **The frontend Dockerfile hardcodes `proxy_pass http://spur-cloud-api:8080`** for K8s/Compose use. In single-BM we don't have a Docker network with that hostname, so we `sed` the nginx config to `host.docker.internal:${SC_API_PORT}` after start (Step 9). A cleaner alternative is a user-defined Docker network with all containers attached; this is a deliberate trade-off for skill simplicity.

### GPU-specific
- **AMD GPUs**: spurd's job runner reads `/dev/kfd` and `/dev/dri/*`. Make sure the user running spurd (root) has rwx access (default on standard ROCm installs).
- **NVIDIA GPUs**: you'll likely want `--gpus all` semantics — but in `bare_metal` mode spurd doesn't shim through Docker, so jobs run as bare processes and inherit the host's NVIDIA driver visibility directly. If the user wants container-isolated GPU sessions, that's a Docker/K8s backend conversation, not bare_metal.

## Report back

End the run with:
- Resolved git ref (e.g. `v0.3.0` or `origin/main → abc123f`)
- `SC_PUBLIC_URL` (where the user opens the UI)
- Container + daemon PIDs/IDs
- For `local` auth: the test account credentials (or "no test user created" if smoke step C was skipped)
- For OAuth: the exact callback URL the user must register with the IdP
- Any deviation from this skill — flag it so the skill can be patched
