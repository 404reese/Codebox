# CodeBox

**A self-hosted, Judge0-compatible code execution engine.**

<small>Forked from [Hitesh Choudhary's CodeBox](https://github.com/hiteshchoudhary/Codebox).</small>

---

## What is CodeBox?

CodeBox is a self-hosted code execution service for coding challenges and practice problems. It securely runs user-submitted code in isolated environments and returns the results - just like LeetCode or HackerRank.

### Key Features

- **Judge0 API Compatible** - Drop-in replacement, works with existing integrations
- **Dual Execution Engines** - Firecracker microVMs (125ms) or Docker containers (500ms)
- **Auto-Detection** - Automatically picks the fastest available executor
- **6 Languages** - Python, JavaScript, TypeScript, C, C++, Java (easily extensible)
- **Batch Submissions** - Run multiple test cases in one request
- **Production Ready** - Redis queue, Prometheus metrics, auto-SSL with Caddy

---

## How It Works

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    Client    │────▶│   CodeBox    │────▶│    Redis     │────▶│   Workers    │
│   Frontend   │     │     API      │     │    Queue     │     │              │
└──────────────┘     └──────────────┘     └──────────────┘     └──────┬───────┘
                                                                      │
                                                    ┌─────────────────┴─────────────────┐
                                                    │                                   │
                                             ┌──────▼──────┐                    ┌───────▼──────┐
                                             │ Firecracker │                    │    Docker    │
                                             │  (microVM)  │        OR          │  (container) │
                                             └──────┬──────┘                    └───────┬──────┘
                                                    │                                   │
                                                    └─────────────┬─────────────────────┘
                                                                  │
                                                    ┌─────────────▼─────────────┐
                                                    │  Sandboxed Code Execution │
                                                    │  (isolated, time-limited) │
                                                    └───────────────────────────┘
```

---

## Running CodeBox

Pick the row that matches your situation.

| Mode | You need | URL you end up with | TLS | Build? |
|------|----------|---------------------|-----|--------|
| **Fastest path** | A server + Docker | `http://<server-ip>` | none | no — pulls from Docker Hub |
| **1. Local** | A laptop with Docker | `http://localhost:3000` | none | yes |
| **2. Server, no domain** | A server + its IP | `http://<server-ip>` | none | yes |
| **3. Server + domain** | A server + a DNS A record | `https://codebox.example.com` | automatic | yes |

If you just want CodeBox running on a VPS and are not changing the code, use
the **fastest path** — one file, no clone, no build. Modes 2 and 3 build from
source and are otherwise identical to each other; the only difference between
them is whether `DOMAIN` is set in `.env`, and you can move from one to the
other later without rebuilding.

---

### Mode 1 — Local development

Runs the API, a worker and Redis on your machine using the Docker executor.
No domain, no reverse proxy, no SSL.

**Prerequisites:** Docker Desktop, Node.js 20+

```bash
git clone https://github.com/404reese/Codebox.git
cd Codebox

npm install

# Build the language runtime images (python, node, gcc, java, ...)
./scripts/build-images.sh

# Start API + worker + Redis
docker compose up -d

curl http://localhost:3000/health
```

The dev stack ships with the token `dev-token` already set:

```bash
curl -X POST "http://localhost:3000/submissions?wait=true" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: dev-token" \
  -d '{"source_code": "print(\"Hello, CodeBox!\")", "language_id": 71}'
```

The API is on port 3000 directly — Caddy is not involved in this mode.

---

### Fastest path — one file, no clone, no build

Prebuilt images are published to Docker Hub, and the production stack needs
**no language images**: the worker image already bundles gcc, JDK 17, Python,
Node and TypeScript, plus NumPy, Matplotlib and pandas. Isolate runs the code
inside that container. So a
whole deployment is one file plus a `.env`.

On a fresh server with Docker installed:

```bash
mkdir -p /opt/codebox && cd /opt/codebox

# Grab the single-file compose from this repository
curl -fsSL https://raw.githubusercontent.com/404reese/Codebox/main/docker-compose.hub.yml \
  -o docker-compose.yml

# Generate secrets and write .env
cat > .env <<EOF
CODEBOX_IMAGE_NAMESPACE=riddheshc
CODEBOX_VERSION=python-libs-v1

AUTH_TOKEN=$(openssl rand -hex 32)
METRICS_TOKEN=$(openssl rand -hex 16)
GRAFANA_PASSWORD=$(openssl rand -hex 16)

# Leave DOMAIN blank for plain HTTP on this server's IP.
# Set it to a domain with an A record here for automatic HTTPS.
DOMAIN=

# Size these to the server. WORKER_CPUS must not exceed its vCPU count.
WORKER_CPUS=1.5
WORKER_MEMORY=4G
WORKER_CONCURRENCY=2
EOF

chmod 600 .env

docker compose pull
docker compose up -d
```

Then check it and grab your token:

```bash
curl http://localhost/health
docker compose ps
docker compose exec worker python3 -c "import numpy, pandas, matplotlib; print('All libraries available')"
grep AUTH_TOKEN .env
```

`docker-compose.hub.yml` carries the Caddy and Prometheus configs inline as
Compose `configs`, so there are genuinely no other files to copy. If you would
rather paste than curl, open the file on GitHub and paste it straight into
`docker-compose.yml` on the server — it needs Docker Compose v2.23 or newer
(`docker compose version`).

To switch an existing deployment to your published images, keep your existing
`.env` secrets and add or update these settings:

```dotenv
CODEBOX_IMAGE_NAMESPACE=riddheshc
CODEBOX_VERSION=python-libs-v1
```

Then pull your images and recreate the API and worker:

```bash
cd /opt/codebox
docker compose pull api worker
docker compose up -d api worker
```

For future upgrades, set `CODEBOX_VERSION` to a tag published for both images
and repeat those commands. The default is `python-libs-v1`; set `latest`
explicitly if you want the latest automated build. Do not rerun the fresh-server
`.env` generation command on an existing deployment, because it replaces secrets.

The GitHub download requires these changes to be pushed to `404reese/Codebox`
on `main`. Until then, copy your local `docker-compose.hub.yml` to the server as
`docker-compose.yml`, or use the upstream Compose file with the `.env` overrides
above. Docker Hub images and GitHub source files are published separately.

> This path uses the prebuilt images. Build from source instead — modes 2 and 3
> below — or build and publish your own updated images when you change the code.

---

### Mode 2 — Server without a domain (IP only, no SSL)

Use this when you have a VPS but no domain yet, or you are running CodeBox on a
private network. CodeBox is served as plain HTTP on port 80 of the server's IP.
**No DNS record and no certificate are required.**

```bash
ssh root@<server-ip>

git clone https://github.com/404reese/Codebox.git /opt/codebox
cd /opt/codebox

# Run setup with NO argument -> IP mode
./scripts/setup-production.sh
```

The script installs Docker, opens the firewall, generates an API token, builds
the images and starts everything. When it finishes it prints your URL and token.

**Doing it by hand instead:**

```bash
cp .env.example .env
# Set AUTH_TOKEN. Leave DOMAIN blank -- that is what selects IP mode.
nano .env

docker compose -f docker-compose.prod.yml up -d --build
```

Verify from your own machine:

```bash
curl http://<server-ip>/health

curl -X POST "http://<server-ip>/submissions?wait=true" \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: <your-token>" \
  -d '{"source_code": "print(2+2)", "language_id": 71}'
```

> **Note the trade-off.** Without TLS, requests and your `X-Auth-Token` cross
> the network in cleartext. That is fine for a private network, an internal
> tool, or a staging box. Do not put a public production frontend on it —
> use mode 3 instead, which is a one-line change.

---

### Mode 3 — Server with a domain and HTTPS

Same stack as mode 2, plus a certificate. Caddy obtains and renews it from
Let's Encrypt automatically — there is no certbot step and no cron job.

**Before you start,** point an `A` record at the server and confirm it resolves:

```bash
dig +short codebox.example.com     # must print your server's IP
```

Ports **80 and 443** must both be reachable from the internet. Port 80 is not
optional — Let's Encrypt uses it to validate the domain. Check your provider's
panel firewall (Hostinger, DigitalOcean, AWS security groups) as well as `ufw`.

```bash
ssh root@<server-ip>

git clone https://github.com/404reese/Codebox.git /opt/codebox
cd /opt/codebox

# Pass the domain as the argument -> HTTPS mode
./scripts/setup-production.sh codebox.example.com
```

**Doing it by hand instead:**

```bash
cp .env.example .env
# Set AUTH_TOKEN, and set DOMAIN=codebox.example.com
nano .env

docker compose -f docker-compose.prod.yml up -d --build
```

Verify:

```bash
curl https://codebox.example.com/health
```

HTTP is redirected to HTTPS automatically.

---

### Switching from mode 2 to mode 3 later

Nothing is rebuilt and no data is lost:

```bash
# 1. Point an A record at the server, wait for it to resolve
dig +short codebox.example.com

# 2. Set the domain
nano .env                 # DOMAIN=codebox.example.com

# 3. Recreate Caddy
docker compose -f docker-compose.prod.yml up -d
```

Caddy picks up the domain, requests a certificate and starts serving HTTPS
within a few seconds. To go back to IP mode, blank out `DOMAIN` and re-run
the same command.

---

### How the two server modes actually differ

One variable. `docker-compose.prod.yml` turns `DOMAIN` into Caddy's site
address:

```yaml
- SITE_ADDRESS=${DOMAIN:-:80}
```

| `DOMAIN` in `.env` | Caddy site address | Result |
|--------------------|--------------------|--------|
| blank or absent | `:80` | Serves HTTP on every IP and hostname. Caddy never requests a certificate and never binds 443. |
| `codebox.example.com` | `codebox.example.com` | Binds 80 and 443, obtains a certificate, redirects HTTP to HTTPS. |

Caddy only attempts certificate issuance when the site address carries a
hostname, so IP mode cannot fail on a DNS or ACME error.

---

### Server sizing

| Load | Specs | Notes |
|------|-------|-------|
| **Light** (< 100 submissions/hr) | 2 vCPU, 4GB RAM | Set `WORKER_CPUS=1.5`, `WORKER_MEMORY=3G` |
| **Medium** (< 1000 submissions/hr) | 4 vCPU, 8GB RAM | Defaults in `.env.example` fit here |
| **Heavy** (> 1000 submissions/hr) | 8 vCPU, 16GB RAM | Raise `WORKER_CPUS`, `WORKER_MEMORY`, `WORKER_CONCURRENCY` |

> `WORKER_CPUS` must never exceed the host's vCPU count — Docker refuses to
> start the container with `Range of CPUs is from 0.01 to N.00`. All resource
> limits are tunable in `.env`; see the block at the bottom of `.env.example`.

> For Firecracker support, choose a host with dedicated vCPUs and nested
> virtualisation (`/dev/kvm` must exist). Most shared VPS plans do not offer
> it — CodeBox falls back to isolate or Docker automatically.

---

## Production Hardening (Optional)

Secure your droplet before deploying. Run these commands as `root`.

### 1. Create Deploy User

```bash
# Create user with sudo access
adduser deploy
usermod -aG sudo deploy
usermod -aG docker deploy

# Copy SSH keys to new user
mkdir -p /home/deploy/.ssh
cp ~/.ssh/authorized_keys /home/deploy/.ssh/
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys
```

### 2. Disable Root & Password Login

```bash
# Edit SSH config
nano /etc/ssh/sshd_config
```

Update these lines:
```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

```bash
# Restart SSH
systemctl restart sshd
```

> **Warning:** Make sure you can login as `deploy` user before disabling root!

### 3. Configure Firewall

```bash
# Allow only essential ports
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp    # SSH
ufw allow 80/tcp    # HTTP
ufw allow 443/tcp   # HTTPS
ufw enable
```

### 4. Enable Automatic Security Updates

```bash
apt install unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades
```

### 5. Install Fail2Ban (Block Brute Force)

```bash
apt install fail2ban
systemctl enable fail2ban
systemctl start fail2ban
```

### 6. Run Setup as Deploy User

```bash
# Login as deploy user
ssh deploy@your-droplet-ip

# Clone and setup
sudo git clone https://github.com/404reese/Codebox.git /opt/codebox
sudo chown -R deploy:deploy /opt/codebox
cd /opt/codebox

# Run production setup
# With a domain (HTTPS):
./scripts/setup-production.sh api.yourdomain.com

# Or without one (plain HTTP on this server's IP):
./scripts/setup-production.sh
```

### Quick Security Checklist

- [ ] Non-root user created (`deploy`)
- [ ] SSH key authentication only
- [ ] Root login disabled
- [ ] Password login disabled
- [ ] Firewall enabled (UFW)
- [ ] Fail2Ban installed
- [ ] Auto-updates enabled

---

## Publishing images (maintainers)

`.github/workflows/publish-images.yml` builds and pushes both images to Docker
Hub on every push to `main`, on every `v*` tag, and on manual dispatch:

| Image | From |
|-------|------|
| `riddheshc/codebox-api` | `docker/api/Dockerfile` |
| `riddheshc/codebox-worker` | `docker/worker/Dockerfile` |

Tags pushed: `latest` (main), the git tag (`v1.2.3`), and the short SHA.

**One-time setup.** Add two repository secrets under
*Settings → Secrets and variables → Actions*:

| Secret | Value |
|--------|-------|
| `DOCKERHUB_USERNAME` | `riddheshc` |
| `DOCKERHUB_TOKEN` | an access token from [hub.docker.com/settings/security](https://hub.docker.com/settings/security), scope **Read & Write** — a token, not your password |

Both repositories must exist on Docker Hub (or the account must allow
auto-creation) before the first push.

Images are built for **linux/amd64 only**. `docker/worker/Dockerfile` hardcodes
`x86_64-linux-gnu` and `java-17-openjdk-amd64`. Check `uname -m` on the VPS:
`x86_64` matches these images; `aarch64` requires changes for ARM. On Apple
Silicon, running these images requires amd64 emulation.

### Build, test and publish manually

Start Docker Desktop with its Linux engine running. From the repository root,
run these commands (also valid in Windows PowerShell):

```bash
docker login --username riddheshc
docker build --platform linux/amd64 -f docker/api/Dockerfile -t riddheshc/codebox-api:python-libs-v1 .
docker build --platform linux/amd64 -f docker/worker/Dockerfile -t riddheshc/codebox-worker:python-libs-v1 .
docker run --rm --entrypoint python3 riddheshc/codebox-worker:python-libs-v1 -c "import numpy, pandas, matplotlib; matplotlib.use('Agg'); import matplotlib.pyplot as plt; plt.plot([1,2,3]); plt.savefig('/tmp/test.png'); print('Libraries imported and plot saved')"
docker push riddheshc/codebox-api:python-libs-v1
docker push riddheshc/codebox-worker:python-libs-v1
```

Use your Docker Hub access token at the login password prompt. For a new release,
replace `python-libs-v1` in both build and push commands with a new tag, then
update `CODEBOX_VERSION` on the server. The automated workflow publishes
`latest`, Git release tags and SHA tags; it does not automatically publish the
manual `python-libs-v1` tag.

---

## API Reference

### Submissions

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/submissions` | Create submission (async) |
| `POST` | `/submissions?wait=true` | Create and wait for result |
| `GET` | `/submissions/:token` | Get result by token |
| `POST` | `/submissions/batch` | Submit multiple (up to 20) |
| `GET` | `/submissions/batch?tokens=a,b,c` | Get multiple results |

### System

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/health` | Health check |
| `GET` | `/languages` | List supported languages |
| `GET` | `/statuses` | List status codes |
| `GET` | `/executor` | Show executor info |
| `GET` | `/metrics` | Prometheus metrics |

---

## Supported Languages

| ID | Language | Image |
|----|----------|-------|
| 50 | C (GCC 9) | `codebox/gcc:9` |
| 54 | C++ (GCC 9) | `codebox/gcc:9` |
| 62 | Java (OpenJDK 17) | `codebox/java:17` |
| 63 | JavaScript (Node 18) | `codebox/node:18` |
| 71 | Python (3.8) | `codebox/python:3.8` |
| 74 | TypeScript (5.0.3) | `codebox/typescript:5` |
| 89 | Multi-file program | `codebox/multi:latest` |

These Judge0 CE language IDs are aliases. A submission with one of them runs the language in the right-hand column.

| Alias ID | Runs as |
|----------|---------|
| 92 | Python (71) |
| 93 | JavaScript (63) |
| 94 | TypeScript (74) |
| 102 | JavaScript (63) |

---

## Status Codes

| ID | Description |
|----|-------------|
| 1 | In Queue |
| 2 | Processing |
| 3 | Accepted |
| 4 | Wrong Answer |
| 5 | Time Limit Exceeded |
| 6 | Compilation Error |
| 7-12 | Runtime Errors |
| 13 | Internal Error |

---

## Important URLs

### Local Development

| Service | URL |
|---------|-----|
| API | http://localhost:3000 |
| Health Check | http://localhost:3000/health |
| Metrics | http://localhost:3000/metrics |

### Production

| Service | URL (mode 2, no domain) | URL (mode 3, with domain) |
|---------|-------------------------|---------------------------|
| API | http://\<server-ip\> | https://your-domain.com |
| Health Check | http://\<server-ip\>/health | https://your-domain.com/health |
| Prometheus | http://localhost:9090 (SSH tunnel) | same |
| Grafana | http://localhost:3001 (SSH tunnel) | same |

Prometheus and Grafana are never published to the internet in either mode —
reach them through an SSH tunnel.

**Access Grafana via SSH tunnel:**
```bash
ssh -L 3001:localhost:3001 user@your-server
# Then open http://localhost:3001
```

---

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `AUTH_TOKEN` | - | **Required.** API authentication token. Compose refuses to start without it. |
| `DOMAIN` | blank | Blank = plain HTTP on the server IP. Set = automatic HTTPS for that domain. |
| `EXECUTOR_TYPE` | `auto` | `auto`, `isolate`, `docker`, or `firecracker` |
| `WORKER_CONCURRENCY` | `4` (dev) / `2` (prod) | Parallel executions per worker |
| `WORKER_CPUS` | `1.5` | Must not exceed the host's vCPU count |
| `WORKER_MEMORY` | `4G` | Worker container memory limit |
| `DEFAULT_CPU_TIME_LIMIT` | `5` | Seconds |
| `DEFAULT_MEMORY_LIMIT` | `128000` | KB |

See `.env.example` for all options.

---

## Security

- **Sandboxed Execution** - Each submission runs in isolation
- **Network Disabled** - No internet access from user code
- **Resource Limits** - CPU, memory, and process limits
- **Time Limits** - Prevents infinite loops
- **Non-root** - Code runs as unprivileged user

---

## Project Structure

```
codebox/
├── src/
│   ├── api/           # Express routes & middleware
│   ├── executor/      # Docker & Firecracker executors
│   ├── queue/         # BullMQ job processing
│   ├── languages/     # Language configurations
│   └── utils/         # Config, logger, helpers
├── docker/
│   └── images/        # Language runtime Dockerfiles
├── scripts/
│   ├── setup.sh                # Local setup
│   ├── setup-production.sh     # Production setup
│   └── build-images.sh         # Build Docker images
├── docker-compose.yml          # Local development
├── docker-compose.prod.yml     # Production with Caddy
└── Caddyfile                   # Auto-SSL config
```

---

## Postman Collections

Import these into Postman for testing:

- `postman_collection.json` - Basic API tests
- `postman_leetcode_tests.json` - LeetCode-style test cases

---

## Contributing

Contributions are welcome. Open an issue or pull request in
[404reese/Codebox](https://github.com/404reese/Codebox).

---

## License

MIT
