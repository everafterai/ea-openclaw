# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Read first: AGENTS.md

This repo is a **fork of OpenClaw** (`everafterai/ea-openclaw`). All upstream OpenClaw conventions — code style, gates, test/CI rules, plugin/channel boundary rules, commit/PR workflow — live in [AGENTS.md](AGENTS.md) and the scoped `AGENTS.md` files under each subtree. Read [AGENTS.md](AGENTS.md) before touching anything in [src/](src/) or [extensions/](extensions/).

@AGENTS.md

This file (`CLAUDE.md`) is fork-specific and covers **deployment context only** — the live agent doesn't run on your laptop, so editing files locally has no effect until they reach EC2.

## Critical deployment fact

**Code does not run locally.** This fork is deployed to an AWS EC2 instance, runs **inside a Docker container** managed by Docker Compose, and that container is what serves the live OpenClaw gateway / Slack bot (Base AI, `B0B307KR0UR`). Any change — code, config, plugin, agent JSON — only takes effect after it has been pushed to the EC2 box and the relevant container has been rebuilt or restarted.

**Implication:** Editing a file in this repo on the laptop **does nothing** to the running agent. Don't claim a fix is live until it has been (a) pushed to EC2, (b) picked up by the gateway (rebuild / restart / hot reload), and (c) verified on the box.

### Deployment topology

| Layer | Where |
|---|---|
| Source of truth | This repo (`everafterai/ea-openclaw`) on GitHub |
| Clone on EC2 | `~/ea-openclaw` on the box |
| Runtime | Docker container `openclaw-gateway` (image built from `Dockerfile.minimus`, a distroless Minimus base — **no shell, no `ls`, no `curl` inside**) |
| Sidecars | `litellm` container (controlled by [`lc`](deploy/bin/lc)), other services per [docker-compose.yml](docker-compose.yml) + [docker-compose.prod.yml](docker-compose.prod.yml) |
| Gateway config (live) | `/home/ubuntu/.openclaw/openclaw.json` on EC2, bind-mounted into the container as `/home/node/.openclaw/openclaw.json`. **This is the file the agent actually reads** — never the one in your laptop's home dir. |
| Gateway port | `ws://127.0.0.1:18789` inside the box (loopback-only by `docker-compose.prod.yml`); reachable from the laptop via SSH tunnel `-L 18789:127.0.0.1:18789` |
| SSH host | See [keys/help.md](keys/help.md) — host changes on EC2 stop/start unless an Elastic IP is attached |
| SSH key | `keys/open-claw.pem` (gitignored) |

### The deployment loop

| Change kind | How it gets to EC2 |
|---|---|
| Source code under [src/](src/), [extensions/](extensions/), [packages/](packages/), `Dockerfile*` | Commit → push to fork → SSH in → `cd ~/ea-openclaw && git pull && dcp build openclaw-gateway && dcp up -d openclaw-gateway` |
| `docker-compose*.yml` or `.env` (env var change) | SSH in → edit → `dcp up -d --force-recreate <service>` (a plain `restart` won't see new env vars — they're baked at container create) |
| `~/.openclaw/openclaw.json` on EC2 (RBAC, agent tiers, channel config, bindings) | SSH in → edit directly **or** use `bai` commands → `bai reload` (full restart, ~10s, required for new agents) or `bai reload --hot` (in-process, routing/binding/access-group changes only) |
| LiteLLM proxy config ([deploy/litellm/config.yaml](deploy/litellm/config.yaml)) | SSH in → `lc edit` (edits + restarts) or edit + `lc restart` |

There is no CI/CD pulling from `main` automatically — every deploy is a manual SSH + `git pull` + rebuild. Treat that as part of the task: a PR merged on GitHub is not a shipped change.

## EC2 shims

Three shell shims sit in front of the deployment. Know what each one does before suggesting commands.

### `dcp` — Docker Compose wrapper (alias)

```
alias dcp='docker compose -f docker-compose.yml -f docker-compose.prod.yml'
```

Defined in [docker-compose.prod.yml:19](docker-compose.prod.yml#L19) tip and installed on EC2 via `.zshrc` per [walkthrough/02-OPENCLAW-DOCKER.md:108-112](walkthrough/02-OPENCLAW-DOCKER.md#L108-L112). **Always use `dcp` on EC2** — `docker compose` without both `-f` flags misses the prod overlay (loopback-only ports, `cap_drop: ALL`, resource limits) and `docker-compose.override.yml` is gitignored upstream so it won't be auto-merged.

Common forms:
- `dcp build openclaw-gateway` — rebuild image after code changes
- `dcp up -d openclaw-gateway` — start/replace the gateway
- `dcp up -d --force-recreate <svc>` — required after `.env` or compose changes (env vars are baked at create)
- `dcp logs -f openclaw-gateway` — follow logs
- `dcp exec openclaw-gateway <cmd>` — run something inside the live container
- `dcp restart openclaw-gateway` — quick restart (config-volume changes only; not for code or env)

### `openclaw` — gateway CLI shim (EC2)

A wrapper at `/usr/local/bin/openclaw` on EC2 that runs `dcp exec openclaw-gateway node dist/index.js "$@"`. Lets upstream OpenClaw docs (`openclaw agents list`, `openclaw plugins enable …`, `openclaw doctor`) work transparently against the deployed gateway. **Footgun**: if you've also installed `openclaw` natively on your laptop, it points at *your laptop's* `~/.openclaw` — a different gateway with different config. To manage the deployed gateway, always run via SSH (or `ssh ec2-claw 'openclaw …'`).

For bootstrap commands that need the gateway *down* (e.g. `openclaw setup`, recovery from a restart loop), use `dcp run --rm openclaw-cli <cmd>` instead — fresh container, same volumes, no dependency on the gateway being healthy.

### `bai` — RBAC / agent-tier config CLI ([deploy/bin/bai](deploy/bin/bai))

Edits `~/.openclaw/openclaw.json` on the EC2 host (the live config the gateway reads via bind-mount). Manages the three-tier RBAC model (admin / ops / view) plus access groups, per-user DM bindings, and `commands.ownerAllowFrom`. Reads/writes JSON atomically (`jq` → tempfile → mv) and validates structure.

Common forms (full reference at the top of the script):
- `bai admin add U… [name]` / `bai operator add U… [name]` / `bai viewer add U… [name]` — tier membership by Slack member ID
- `bai whois U…` / `bai tiers` — read current state
- `bai validate` — JSON + structural sanity check
- `bai reload` — validate + full `dcp restart openclaw-gateway` (slow but reliable; **required after adding/removing agents**)
- `bai reload --hot` — validate + `openclaw config reload` (in-process; routing/access-group/binding changes only, **does not re-init new agents**)
- `bai migrate` — one-shot fixup for older skeleton shape (renames `base-* → *`, fixes `botToken/appToken/ownerDisplaySecret` shapes, rewrites `tools "memory" → "group:memory"`, etc.)
- `bai bootstrap` — runs `openclaw agents add` for admin/ops/view to create workspace bootstrap files (required after `bai migrate` before UI shows the agents)
- `bai path` — print resolved config path

Env vars: `BAI_CONFIG` (override config path), `BAI_NAMES` (sidecar UID→name map path), `BAI_COMPOSE_DIR` (clone dir for reload fallback, default `$HOME/ea-openclaw`).

Design spec: `docs/superpowers/specs/2026-05-10-openclaw-slack-rbac-design.md`.

### `lc` — LiteLLM control shim ([deploy/bin/lc](deploy/bin/lc))

Manages the dockerized LiteLLM sidecar (container default `ea-openclaw-litellm-1`, override via `LITELLM_CONTAINER`). Auto-injects the master key from `.env` for HTTP helpers.

Common forms:
- `lc` / `lc ps` — container status
- `lc logs [args]` — follow logs
- `lc restart` / `lc reload` — restart (config-file changes only)
- `lc recreate` — `dcp up -d --force-recreate litellm` (required for `.env` / compose changes)
- `lc edit` — edit [deploy/litellm/config.yaml](deploy/litellm/config.yaml) then restart
- `lc shell` — shell into the container
- `lc health` / `lc ready` / `lc models` / `lc test <model> [msg]` — HTTP helpers (master key auto-injected)
- `lc spend [today|month|range <s> <e>|report|logs|raw]` — spend tracking (requires `DATABASE_URL` in compose)
- `lc curl <path> [args]` — arbitrary authenticated GET against the proxy

## Operational guardrails specific to this fork

- **The Minimus image is distroless.** Don't suggest `dcp exec openclaw-gateway bash` / `sh` / `ls` / `cat` / `curl` — those binaries are not in the image. Probe via full path (`dcp exec openclaw-gateway /usr/local/bin/gog --help`) or attach a debug sidecar with shared volumes: `docker run --rm -it --volumes-from $(dcp ps -q openclaw-gateway) debian:12-slim bash`.
- **The `openclaw.json` you find on the laptop is almost certainly not the live config.** A leftover skeleton may exist (e.g. under `/private/tmp/...`) — verify on EC2 with `bai path` or by inspecting the running container's bind mount before claiming a config fix is applied.
- **Slack credentials must not leak via `process.env` to agent shells.** Per the upstream [AGENTS.md](AGENTS.md), channel creds belong in `~/.openclaw/credentials/` behind the plugin, not as bare `SLACK_BOT_TOKEN` in the gateway's environment. If an agent has `exec` enabled *and* the token in env *and* no native `message` tool in its allowlist, it will reinvent the Slack API via `curl` — the exact failure mode that motivated this fork's RBAC tightening.
- **Agent tool allowlists scale via profiles + groups, not per-tool lists.** Use `tools.profile: "coding" | "messaging" | "minimal" | "full"` with `alsoAllow` for extras and `deny` for cutouts. Available group expansions are defined in [src/agents/tool-catalog.ts](src/agents/tool-catalog.ts) (`buildCoreToolGroupMap` → `group:fs`, `group:web`, `group:memory`, `group:messaging`, `group:plugins`, etc.). `allow` + `alsoAllow` in the same scope is a schema error ([src/config/zod-schema.agent-runtime.ts:631-637](src/config/zod-schema.agent-runtime.ts#L631-L637)).
- **`docker compose v2.20+` is required** for the `!override` list-replace tags used by [docker-compose.prod.yml](docker-compose.prod.yml).
- **`docker group is root-equivalent** on the EC2 box. Anyone with shell as `ubuntu` can mount `/` into a container. Keep the SG locked to your IP; rotate `keys/open-claw.pem` if it leaks.

## Fork-specific files worth knowing

| File | Purpose |
|---|---|
| [Dockerfile.minimus](Dockerfile.minimus) | Thin child of Minimus's hardened distroless OpenClaw base; copies `gog` + `goplaces` skill binaries in with checksum verification |
| [docker-compose.prod.yml](docker-compose.prod.yml) | Prod overlay — loopback-only port bindings, `cap_drop: ALL`, resource limits |
| [.env.example](.env.example) | Operator template; "Deploy / docker-compose" section at the bottom holds `OPENCLAW_GATEWAY_TOKEN`, `GOG_KEYRING_PASSWORD`, `OPENCLAW_CONFIG_DIR`, `OPENCLAW_WORKSPACE_DIR` |
| [deploy/bin/bai](deploy/bin/bai), [deploy/bin/lc](deploy/bin/lc) | Operator shims (above) |
| [deploy/litellm/config.yaml](deploy/litellm/config.yaml) | LiteLLM proxy model routing |
| [walkthrough/](walkthrough/) | Step-by-step EC2 + Docker + Gemini/Vertex provisioning. Read before suggesting infra changes. |
| [keys/help.md](keys/help.md) | Current EC2 SSH target (DNS changes on stop/start) |
| [EA-README.md](EA-README.md) | Fork-level README — installs, SSH access, stop/start, EBS resize, update loop |

## Common laptop ↔ EC2 mismatches to watch for

When debugging, always state which side you mean:

| Symptom | Likely confusion |
|---|---|
| "I edited the config but nothing changed" | Edited the laptop's `~/.openclaw/openclaw.json` (or a `/tmp/` fixture), not EC2's `/home/ubuntu/.openclaw/openclaw.json` |
| "`openclaw agents list` shows different agents on EC2 vs my laptop" | You have OpenClaw installed natively on the laptop too — it's a different gateway with a different config volume |
| "The bot used `curl` instead of a Slack tool" | Live agent's `tools.allow` on EC2 lacks `message` (the cross-channel send tool); local repo's config is irrelevant |
| "`.env` change didn't apply after `dcp restart`" | Env vars are baked at container create — need `dcp up -d --force-recreate <svc>` |
| "Slack token is in `env` — why?" | Inbound Slack channel plugin reads `SLACK_BOT_TOKEN` from env (manifest at [extensions/slack/openclaw.plugin.json](extensions/slack/openclaw.plugin.json) declares `channelEnvVars`). Move to `~/.openclaw/credentials/` to close the `exec`+`curl` fallback path. |
