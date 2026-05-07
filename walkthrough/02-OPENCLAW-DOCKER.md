# 02 — OpenClaw on EC2 (Docker)

**Run this on the EC2 instance** (SSH in first). Nothing runs on your local machine.

**Reference:** [OpenClaw docs](https://docs.openclaw.ai/)

---

## 1. SSH into EC2

Use the public DNS from step 01:

```bash
ssh -i keys/open-claw.pem ubuntu@<EC2_PUBLIC_DNS>
```

If connection refused, wait 1–2 min after instance launch.

## 2. Install Docker

```bash
sudo apt-get update
sudo apt-get install -y git curl ca-certificates
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
```

Log out and back in for the group change:

```bash
exit
```

Then SSH in again:

```bash
ssh -i keys/open-claw.pem ubuntu@<EC2_PUBLIC_DNS>
```

Verify:

```bash
docker --version
docker compose version
```

## 2a. zsh + oh-my-zsh + zsh-autosuggestions (optional)

```bash
sudo apt-get install -y zsh
CHSH=yes RUNZSH=no sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

Edit `~/.zshrc` and add `zsh-autosuggestions` to the plugins line:

```bash
nano ~/.zshrc
```

Change `plugins=(git)` to `plugins=(git zsh-autosuggestions)`.

Set zsh as default shell:

```bash
chsh -s $(which zsh)
```

Log out and SSH back in for it to take effect.

## 3. Clone OpenClaw

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
```

## 4. Create persistent dirs

```bash
mkdir -p ~/.openclaw
mkdir -p ~/.openclaw/workspace
```

## 5. Create `.env`

```bash
# Generate secrets
openssl rand -hex 32
# Use output for tokens below

nano .env
```

Paste (replace secrets with output from `openssl rand -hex 32`):

```
OPENCLAW_IMAGE=openclaw:local
OPENCLAW_GATEWAY_TOKEN=<your-token>
OPENCLAW_GATEWAY_BIND=lan
OPENCLAW_GATEWAY_PORT=18789
OPENCLAW_CONFIG_DIR=/home/$USER/.openclaw
OPENCLAW_WORKSPACE_DIR=/home/$USER/.openclaw/workspace
GOG_KEYRING_PASSWORD=<your-secret>
XDG_CONFIG_HOME=/home/node/.openclaw
```
The base `docker-compose.yml` may reference `CLAUDE_AI_SESSION_KEY`, `CLAUDE_WEB_SESSION_KEY`, `CLAUDE_WEB_COOKIE` — add those to `.env` only if using Claude integration.

**Do not commit `.env`.** Add it to `.gitignore` if not already.

## 6. Docker Compose override

The repo already has `docker-compose.yml`. Create `docker-compose.override.yml` to add build, env file, and loopback-only binding (the gateway is reached over the SSH tunnel, never directly exposed):

```bash
nano docker-compose.override.yml
```

```yaml
services:
  openclaw-gateway:
    build: .
    env_file:
      - .env
    environment:
      GOG_KEYRING_PASSWORD: ${GOG_KEYRING_PASSWORD}
      XDG_CONFIG_HOME: /home/node/.openclaw
    ports:
      - "127.0.0.1:${OPENCLAW_GATEWAY_PORT:-18789}:18789"
      - "127.0.0.1:${OPENCLAW_BRIDGE_PORT:-18790}:18790"
```

This merges with the base config.

## 7. Dockerfile changes

Edit the repo's `Dockerfile` and add the following **after** the `OPENCLAW_DOCKER_APT_PACKAGES` block (after the `fi`), **before** `COPY package.json`:

```dockerfile
# Skills: socat + gog, goplaces (Gmail, Places). gog repo is gogcli.
RUN apt-get update && apt-get install -y socat && rm -rf /var/lib/apt/lists/*
RUN curl -L -o /tmp/gog.tar.gz https://github.com/steipete/gogcli/releases/download/v0.9.0/gogcli_0.9.0_linux_amd64.tar.gz \
  && tar -xzf /tmp/gog.tar.gz -C /usr/local/bin && chmod +x /usr/local/bin/gog && rm /tmp/gog.tar.gz
RUN curl -L -o /tmp/goplaces.tar.gz https://github.com/steipete/goplaces/releases/download/v0.2.1/goplaces_0.2.1_linux_amd64.tar.gz \
  && tar -xzf /tmp/goplaces.tar.gz -C /usr/local/bin && chmod +x /usr/local/bin/goplaces && rm /tmp/goplaces.tar.gz
```

**Note:** wacli (WhatsApp) has no Linux release in v0.2.0; omit if not needed. gog is from the `gogcli` repo.

**Alternative:** Use `--build-arg OPENCLAW_DOCKER_APT_PACKAGES="socat"` and add only the two `RUN curl ...` lines.

## 8. Build and run

On t3.small (2GB RAM), `pnpm install` can OOM during build. Add swap first:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

To persist swap across reboots: `echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab`

Then build:

```bash
docker compose build
docker compose up -d openclaw-gateway
```

Verify binaries:

```bash
docker compose exec openclaw-gateway which gog
docker compose exec openclaw-gateway which goplaces
```

Expected: `/usr/local/bin/gog`, `/usr/local/bin/goplaces`

## 9. Check logs

```bash
docker compose logs -f openclaw-gateway
```

Success: `[gateway] listening on ws://0.0.0.0:18789`

## 10. Access from laptop (SSH tunnel)

On your local machine:

```bash
ssh -i keys/open-claw.pem -L 18789:127.0.0.1:18789 ubuntu@<EC2_PUBLIC_DNS>
```

Run from the project root (where `keys/open-claw.pem` lives). The host is in `keys/help.md`.

Browser: http://127.0.0.1:18789/ — paste gateway token.

---

## 11. Hardening (optional)

The walkthrough above gets you running but is not hardened. The cheap wins:

### 11a. Verify the container's runtime user

```bash
docker compose exec openclaw-gateway id
```

If it prints `uid=0(root)`, the image runs as root. The OpenClaw image typically runs as the `node` user (uid 1000). If it doesn't, pin it explicitly in the override (next step).

### 11b. Drop capabilities and block privilege escalation

Edit `~/openclaw/docker-compose.override.yml` and extend the `openclaw-gateway` service:

```yaml
services:
  openclaw-gateway:
    # ...existing keys...
    user: "1000:1000"          # match the image's non-root user; adjust if `id` showed something else
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    read_only: true
    tmpfs:
      - /tmp
      - /run
```

Then `docker compose up -d` and re-test. If something breaks, remove `read_only: true` first and re-add capabilities one at a time (`cap_add: [NET_BIND_SERVICE]` etc.) only as needed.

### 11c. Pin and checksum the skill binaries

Replace the unchecked downloads in section 7 with verified ones. Compute the checksums once on a trusted machine:

```bash
curl -L https://github.com/steipete/gogcli/releases/download/v0.9.0/gogcli_0.9.0_linux_amd64.tar.gz | sha256sum
curl -L https://github.com/steipete/goplaces/releases/download/v0.2.1/goplaces_0.2.1_linux_amd64.tar.gz | sha256sum
```

Then bake them into the Dockerfile:

```dockerfile
ARG GOG_SHA256=<paste-from-above>
ARG GOPLACES_SHA256=<paste-from-above>

RUN apt-get update && apt-get install -y socat && rm -rf /var/lib/apt/lists/*

RUN curl -fsSL -o /tmp/gog.tar.gz https://github.com/steipete/gogcli/releases/download/v0.9.0/gogcli_0.9.0_linux_amd64.tar.gz \
  && echo "${GOG_SHA256}  /tmp/gog.tar.gz" | sha256sum -c - \
  && tar -xzf /tmp/gog.tar.gz -C /usr/local/bin && chmod +x /usr/local/bin/gog && rm /tmp/gog.tar.gz

RUN curl -fsSL -o /tmp/goplaces.tar.gz https://github.com/steipete/goplaces/releases/download/v0.2.1/goplaces_0.2.1_linux_amd64.tar.gz \
  && echo "${GOPLACES_SHA256}  /tmp/goplaces.tar.gz" | sha256sum -c - \
  && tar -xzf /tmp/goplaces.tar.gz -C /usr/local/bin && chmod +x /usr/local/bin/goplaces && rm /tmp/goplaces.tar.gz
```

Build will fail loudly if the upstream tarball ever changes.

### 11d. Resource limits

Stops a runaway container from taking down the host:

```yaml
    deploy:
      resources:
        limits:
          cpus: "1.5"
          memory: 1500M
```

(Compose v2 honors `deploy.resources.limits` outside Swarm mode.)

### 11e. Caveat: `docker` group on the host is root-equivalent

`sudo usermod -aG docker $USER` (section 2) means anyone who gets a shell as `ubuntu` can mount `/` into a container and become host-root. The container hardening above doesn't change that. Mitigations: keep the SSH security group locked to your IP, use SSH key auth only (default on the AMI), and don't share the key.

---

**Next:** [03-VERTEX-GEMINI.md](./03-VERTEX-GEMINI.md)
