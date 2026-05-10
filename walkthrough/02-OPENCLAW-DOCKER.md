# 02 — OpenClaw on EC2 (Minimus image)

**Run this on the EC2 instance** (SSH in first). Nothing runs on your local machine.

This walkthrough uses the fork (`everafterai/ea-openclaw`), which ships:

- `Dockerfile.minimus` — thin child of [Minimus's hardened distroless OpenClaw](https://www.minimus.io/post/stop-running-openclaw-with-2-000-vulnerabilities-why-minimus-openclaw-image-has-99-fewer-cves) (~7 CVEs vs ~2,000 in the official image), with `gog` + `goplaces` skill binaries copied in (checksumed).
- `docker-compose.prod.yml` — production overlay: loopback-only port bindings, `cap_drop: ALL`, resource limits.
- `.env.example` — operator template (deploy section is at the bottom).

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

Then SSH in again. Verify:

```bash
docker --version
docker compose version       # must be v2.20+ (for `!override` list-replace tags)
```

## 2a. zsh + oh-my-zsh + zsh-autosuggestions (optional)

```bash
sudo apt-get install -y zsh
CHSH=yes RUNZSH=no sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

Edit `~/.zshrc`, change `plugins=(git)` to `plugins=(git zsh-autosuggestions)`. Then:

```bash
chsh -s $(which zsh)
```

Log out and SSH back in.

## 3. Clone the fork

```bash
git clone https://github.com/everafterai/ea-openclaw.git ~/ea-openclaw
cd ~/ea-openclaw
```

Use the SSH form (`git@github.com:everafterai/ea-openclaw.git`) if you've added an SSH key to GitHub from this VM.

## 4. Persistent dirs

```bash
mkdir -p ~/.openclaw ~/.openclaw/workspace
```

These are bind-mounted into the container as `/home/node/.openclaw` and the workspace.

## 5. Configure `.env`

```bash
cp .env.example .env
nano .env
```

At minimum, set these in the **Deploy / docker-compose** section near the bottom:

```
OPENCLAW_GATEWAY_TOKEN=<openssl rand -hex 32>
GOG_KEYRING_PASSWORD=<openssl rand -hex 32>
OPENCLAW_CONFIG_DIR=/home/ubuntu/.openclaw
OPENCLAW_WORKSPACE_DIR=/home/ubuntu/.openclaw/workspace
```

Generate the two random secrets:

```bash
openssl rand -hex 32
openssl rand -hex 32
```

Set model provider API keys you'll use (see `walkthrough/03-VERTEX-GEMINI.md` for Gemini specifics). `.env` is gitignored — never commit it.

## 6. Compose alias

`docker-compose.prod.yml` is **not** auto-merged (the auto-merge name `docker-compose.override.yml` is gitignored by upstream openclaw and reserved for local-machine tweaks). Always invoke with both `-f` flags. One-time alias:

```bash
echo "alias dcp='docker compose -f docker-compose.yml -f docker-compose.prod.yml'" >> ~/.zshrc
source ~/.zshrc
```

(Use `~/.bashrc` if you skipped section 2a.)

## 6a. `openclaw` CLI shim (recommended)

OpenClaw docs throughout reference commands like `openclaw agents list`, `openclaw onboard`, `openclaw plugins enable …`. That `openclaw` binary is the native CLI — installed via `npm i -g openclaw`, the macOS installer, etc. Your EC2 install is fully containerized; the binary lives **inside the image**, not on the host PATH, so docs commands don't work as-is.

Drop a shim script that wraps the running gateway container with `docker compose exec`. Same image as the cli sidecar, but no per-invocation container spin-up — `exec` runs the command **inside the already-running gateway**, which is essentially instant:

```bash
sudo tee /usr/local/bin/openclaw >/dev/null <<'EOF'
#!/usr/bin/env bash
exec docker compose -f ~/ea-openclaw/docker-compose.yml -f ~/ea-openclaw/docker-compose.prod.yml exec openclaw-gateway node dist/index.js "$@"
EOF
sudo chmod +x /usr/local/bin/openclaw
```

Now `openclaw agents list`, `openclaw models set …`, `openclaw plugins enable …` all work, routed transparently into the running gateway. Won't be functional until §7 builds + starts the gateway, but the script itself is fine to lay down now.

**For bootstrap commands** that must run *without* a healthy gateway (`openclaw setup`, recovery commands when the gateway is in a restart loop), use `dcp run --rm openclaw-cli <cmd>` instead — it spins up a fresh container that mounts the same volumes but doesn't depend on the gateway being up. The `dcp run` form costs ~1-2s per call (container create/destroy), so reserve it for bootstrap; for everyday CLI use, `exec` via the shim is faster.

If you'd rather skip `sudo`, an alias does the same job for your shell only:

```bash
echo "alias openclaw='dcp exec openclaw-gateway node dist/index.js'" >> ~/.zshrc
source ~/.zshrc
```

(Aliases occasionally trip on quoted args; the script handles those cleanly.)

**Footgun.** The `openclaw` you'd type on your **laptop** (if you've installed it natively) is a different binary pointing at *your laptop's* `~/.openclaw` — a separate gateway with separate config. The shim above only resolves on EC2 against the deployed gateway's config volume. Don't conflate them: to manage the deployed gateway, always run commands on the EC2 box (or `ssh ec2-claw 'openclaw agents list'` from your laptop).

## 7. Build and run

```bash
dcp build openclaw-gateway
dcp up -d openclaw-gateway
```

Build is fast (~30s) — the Minimus image is prebuilt; the only work is fetching it and copying `gog` + `goplaces` in. **No swap needed** (no `pnpm install` at build time).

If you want to add a local-only tweak that won't be committed (e.g. machine-specific volume mounts, debug flags), drop a `docker-compose.override.yml` next to `docker-compose.prod.yml`. Compose auto-merges it; it's gitignored upstream.

## 8. Verify the binaries

The Minimus image is distroless — no shell, no `which`, no `ls`. Probe the binaries directly by full path:

```bash
dcp exec openclaw-gateway /usr/local/bin/gog --help
dcp exec openclaw-gateway /usr/local/bin/goplaces --help
```

If either errors with "executable file not found", the COPY step in `Dockerfile.minimus` either failed or landed somewhere other than `/usr/local/bin`. For deeper poking, attach a debug sidecar that shares the gateway's volumes:

```bash
docker run --rm -it --volumes-from $(dcp ps -q openclaw-gateway) debian:12-slim bash
```

The most authoritative check is the gateway's own startup log — if openclaw discovers and registers the skills, you'll see them mentioned in section 9's logs.

## 9. Check logs

```bash
dcp logs -f openclaw-gateway
```

Success: `[gateway] listening on ws://0.0.0.0:18789`.

## 10. Access from laptop (SSH tunnel)

On your local machine, from the project root (where `keys/open-claw.pem` lives — host is in `keys/help.md`):

```bash
ssh -i keys/open-claw.pem -L 18789:127.0.0.1:18789 ubuntu@<EC2_PUBLIC_DNS>
```

Browser: http://127.0.0.1:18789/ — paste gateway token.

---

## 11. Optional further hardening

The fork already bakes in: `cap_drop: ALL`, `no-new-privileges`, loopback-only ports, resource limits, checksum-verified binary downloads, multi-arch-pinned Minimus base. Cheap wins on top:

### 11a. Verify the runtime user

```bash
dcp exec openclaw-gateway id
```

Minimus images run as non-root by default. If `id` shows `uid=0(root)`, pin a non-root user explicitly via `user: "1000:1000"` in `docker-compose.prod.yml`.

### 11b. Read-only root filesystem

`docker-compose.prod.yml` ships with `read_only: true` and `tmpfs` commented out (a known footgun until you've confirmed startup works). To enable:

```bash
nano docker-compose.prod.yml   # uncomment the read_only + tmpfs block
dcp up -d --force-recreate openclaw-gateway
dcp logs -f openclaw-gateway
```

If startup fails, comment them out again and re-test.

### 11c. Refresh the Minimus base digest

The base in `Dockerfile.minimus` is pinned to a multi-arch manifest digest. To update:

```bash
docker buildx imagetools inspect us-docker.pkg.dev/prod-375107/minimus-public/openclaw:latest
```

Replace the `@sha256:...` in `Dockerfile.minimus` with the top-level `Digest:`. Commit, push, redeploy.

### 11d. Move secrets off-disk

`.env` on the EC2 box is fine for solo / small-team use. For real internal-prod, move `OPENCLAW_GATEWAY_TOKEN` and `GOG_KEYRING_PASSWORD` to AWS Secrets Manager or SSM Parameter Store and inject at container start (via an entrypoint wrapper or `docker run --env-file=<(aws ssm get-parameter ...)`).

### 11e. Auth proxy in front of the gateway

The gateway token is a shared secret. SSH-tunnel-per-user doesn't scale to "several internal users." For multi-user prod, put one of these in front of the gateway: Cloudflare Access, Tailscale + ACLs, or an ALB with Okta/SAML. This is the highest-leverage security change once user count grows past 2–3.

### 11f. Caveat: `docker` group is root-equivalent

`sudo usermod -aG docker $USER` (section 2) means anyone with shell as `ubuntu` can mount `/` into a container and become host-root. Container hardening doesn't change this. Mitigations: keep the SSH security group locked to your IP, use SSH key auth only (default on the AMI), don't share the key, and rotate it if it leaks.

---

**Next:** [03-VERTEX-GEMINI.md](./03-VERTEX-GEMINI.md)
