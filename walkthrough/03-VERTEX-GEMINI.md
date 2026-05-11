# 03 — Gemini / Vertex AI on AWS EC2

**On AWS EC2 there is no GCP metadata server**, so Application Default Credentials don't auto-resolve. You also can't (currently) make OpenClaw's `google-vertex` provider work directly — its bundled `gaxios` 7.1.4 + `google-auth-library` combo crashes Node's CJS→ESM translator on the dynamic fetch import (see [GH openclaw/openclaw#9729](https://github.com/openclaw/openclaw/issues/9729)). We confirmed the crash on Node 24 and Node 25; it's a package interop bug, not a Node bug.

This walkthrough has two recommended paths and one explicitly-not-working path.

| Option | What | When to use |
|---|---|---|
| **A. Gemini API key** | Direct call to `generativelanguage.googleapis.com` via the `google` provider | Simplest. No Vertex features needed. |
| **B. Vertex via LiteLLM sidecar** | Self-hosted [LiteLLM](https://github.com/BerriAI/litellm) container does Vertex auth in Python; OpenClaw talks to it over HTTP | You want Vertex features (grounding, context caching, IAM-gated models, GCP billing/credits) |
| ~~C. Vertex direct~~ | OpenClaw's `google-vertex` provider | **Broken.** Don't try; you will lose hours. |

**References:**
- [OpenClaw model providers](https://docs.openclaw.ai/concepts/model-providers)
- [LiteLLM + OpenClaw integration](https://docs.litellm.ai/docs/tutorials/openclaw_integration)
- [Vertex AI authentication](https://cloud.google.com/vertex-ai/docs/authentication)

---

## Option A: Gemini API key (simplest)

**Provider:** `google` · **Auth:** `GEMINI_API_KEY` env var

Get a key from [aistudio.google.com/app/apikey](https://aistudio.google.com/app/apikey).

```bash
# On EC2
cd ~/ea-openclaw
echo "GEMINI_API_KEY=<your-key>" >> .env
dcp up -d --force-recreate openclaw-gateway

openclaw models set google/gemini-2.5-pro    # or google/gemini-2.5-flash
openclaw agent --agent main --message "Hello"
```

That's the entire setup. If you don't need Vertex-specific features, stop here.

---

## Option B: Vertex via LiteLLM sidecar (recommended for Vertex)

This is the tested-working path on this stack. Architecture:

```
openclaw-gateway  ──HTTP──▶  litellm (sidecar)  ──HTTPS──▶  Vertex AI
   (Node, OpenClaw)            (Python, in-VPC)              (Google)
```

LiteLLM is run from the upstream OSS image; **no third-party data path** — it runs on your EC2 in the same docker network as the gateway. Not a subprocessor in the legal sense.

The compose file (`docker-compose.prod.yml`) and proxy config (`deploy/litellm/config.yaml`) are already in this repo. Steps below assume you're starting on a fresh EC2 with walkthrough 02 already done.

### B.1. Create the service account (one-time, from your laptop)

Skip if `keys/ea-agent-sa.json` already exists. The `keys/` dir is gitignored.

```bash
PROJECT_ID=<your-gcp-project>          # e.g. everafter-19954

gcloud iam service-accounts create ea-agent-sa \
  --display-name="ea-agent EC2" \
  --project="$PROJECT_ID"

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:ea-agent-sa@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user"

gcloud iam service-accounts keys create keys/ea-agent-sa.json \
  --iam-account="ea-agent-sa@$PROJECT_ID.iam.gserviceaccount.com"

gcloud services enable aiplatform.googleapis.com --project="$PROJECT_ID"
```

### B.2. Upload the SA key to EC2

```bash
# From your laptop, project root
scp -i keys/open-claw.pem keys/ea-agent-sa.json ubuntu@<EC2_PUBLIC_DNS>:~/ea-agent-sa.json

# IMPORTANT: chmod 644, not 600. The LiteLLM container runs as a non-root
# user with a different UID than the host's `ubuntu` user. With 600 the
# container can't read the file (we hit `PermissionError: [Errno 13]`).
# 644 is fine here — the EC2 lives in a gated VPC and only `ubuntu` has
# shell access.
ssh -i keys/open-claw.pem ubuntu@<EC2_PUBLIC_DNS> 'chmod 644 ~/ea-agent-sa.json'
```

**Treat the JSON as a secret** even with 644 — never commit, never paste, rotate annually.

### B.3. Set env vars in `.env` on EC2

```bash
ssh -i keys/open-claw.pem ubuntu@<EC2_PUBLIC_DNS>
cd ~/ea-openclaw

# LiteLLM master key — random, kept on the host. OpenClaw uses this same
# value to authenticate to LiteLLM.
echo "LITELLM_MASTER_KEY=sk-$(openssl rand -hex 24)" >> .env

# Optional overrides; defaults are everafter-19954 / us-central1
# Use `global` if your project's models are at the global location.
echo "GCP_PROJECT_ID=<your-gcp-project>" >> .env
echo "GCP_LOCATION=global"               >> .env

# Sanity-check
grep -E '^(LITELLM_MASTER_KEY|GCP_)' .env
```

> ⚠️ **The env-var key matters.** The compose file reads `GCP_LOCATION`, **not** `GOOGLE_CLOUD_LOCATION`. If you write the wrong key in `.env` the substitution silently falls back to `us-central1` and you get 404s for models that exist only at `global`.

### B.4. Bring up the stack

```bash
git pull                           # pull the docker-compose.prod.yml + deploy/ files
mkdir -p workspace-skills          # bind-mount target for agent-authored skills

dcp up -d --force-recreate         # NOT `dcp restart` — see "Pitfalls" below
```

`docker-compose.prod.yml` builds from the standard `Dockerfile` (Node 24-bookworm-slim). **Don't switch to `Dockerfile.minimus`** — the upstream Minimus base image currently ships Node 25, which has an ESM/CJS translator regression on top of the gaxios bug. We tested both Node versions; both crash for direct Vertex anyway, but the LiteLLM path needs Node 24 at the gateway.

### B.5. Verify LiteLLM is healthy

The `lc` shim (in `deploy/bin/lc`) wraps `docker exec` against the running litellm container.

```bash
# One-time install
echo 'alias lc="$HOME/ea-openclaw/deploy/bin/lc"' >> ~/.zshrc && source ~/.zshrc
# Or symlink onto PATH:
sudo ln -sf $HOME/ea-openclaw/deploy/bin/lc /usr/local/bin/lc

# Probes
lc ps                                 # container status — should be "healthy"
lc health                             # GET /health/liveliness — "I'm alive!"
lc models                             # what aliases the proxy serves
lc test gemini-2.5-pro "Say hi in 3 words"
```

If `lc test` returns Gemini text, **LiteLLM↔Vertex is fully working**. Anything that fails after this point is on the OpenClaw side, not Vertex.

### B.6. Wire OpenClaw to LiteLLM

OpenClaw needs three things in `~/.openclaw/openclaw.json`:

1. `models.providers.litellm.baseUrl` = `http://litellm:4000` (Docker service name, **not** `localhost`)
2. `models.providers.litellm.apiKey` = the `LITELLM_MASTER_KEY` from `.env`
3. `models.providers.litellm.models` = entries with `id`s matching `lc models`

The `openclaw configure` wizard writes a placeholder Claude entry — replace it. Easiest is direct edit:

```bash
KEY=$(grep '^LITELLM_MASTER_KEY=' .env | cut -d= -f2)

nano $(openclaw config file)
```

Replace (or add) the `models` block:

```json
  "models": {
    "mode": "merge",
    "providers": {
      "litellm": {
        "baseUrl": "http://litellm:4000",
        "apiKey": "PASTE_THE_KEY_HERE",
        "api": "openai-completions",
        "models": [
          { "id": "gemini-3-pro-preview",  "name": "Gemini 3 Pro Preview (Vertex)" },
          { "id": "gemini-2.5-pro",        "name": "Gemini 2.5 Pro (Vertex)" },
          { "id": "gemini-2.5-flash",      "name": "Gemini 2.5 Flash (Vertex)" },
          { "id": "gemini-2.0-flash",      "name": "Gemini 2.0 Flash (Vertex)" }
        ]
      }
    }
  }
```

> Each `id` must exactly match a `model_name` entry in `deploy/litellm/config.yaml`, which in turn must match a real Vertex publisher model id. Vertex ids do NOT use a `.x` minor version: it's `gemini-3-pro-preview`, **not** `gemini-3.1-pro-preview`. Verify via [GCP Model Garden](https://console.cloud.google.com/vertex-ai/model-garden) or `lc models`.

Also remove any stale `google-vertex/...` entry from `agents.defaults.models` — it triggers a startup load of the broken `google-vertex` provider, which fails with "No API key found for provider google-vertex" even though you're not using it.

```json
  "agents": {
    "defaults": {
      "model": { "primary": "litellm/gemini-3-pro-preview" },
      "models": {
        "litellm/gemini-3-pro-preview": {}
        // ✗ DELETE any "google-vertex/..." line here
      }
    }
  }
```

Then:

```bash
openclaw models set litellm/gemini-3-pro-preview
openclaw agent --agent main --message "Hello — what model are you?"

# In another shell, watch LiteLLM see the request
docker logs -f ea-openclaw-litellm-1 --tail 0
# expect: POST /v1/chat/completions ... 200 OK
```

If you get Gemini text → done.

---

## Option C: ~~Vertex direct~~ (broken)

OpenClaw's `google-vertex` provider doesn't currently work on this stack:

- Crash: `TypeError: Cannot convert undefined or null to object` at `node:internal/modules/esm/translators` when `gaxios` does `await import(...)` for its fetch implementation.
- Reproduces on Node 24 and Node 25.
- Reproduces with both service-account JSON and user-flow ADC (`gcloud auth application-default login`).
- Tracked in [GH openclaw/openclaw#9729](https://github.com/openclaw/openclaw/issues/9729) — the issue is closed as "implemented" but the underlying bug persists in 2026.4.x and 2026.5.x.

If you must have Vertex, use **Option B**. The LiteLLM proxy uses its own (Python) `google-auth` stack and never executes the broken codepath.

---

## Operating LiteLLM (the `lc` shim)

`deploy/bin/lc` wraps the running container. Common ops:

| Command | What |
|---|---|
| `lc` / `lc ps` | Container status |
| `lc logs -f --tail 80` | Stream logs |
| `lc health` / `lc ready` | Health probes |
| `lc models` | List aliases the proxy serves |
| `lc test <id> [msg]` | One-shot chat completion |
| `lc edit` | Edit `deploy/litellm/config.yaml` and auto-restart |
| `lc reload` / `lc restart` | `docker restart` (config-file changes only) |
| `lc recreate` | `dcp up -d --force-recreate litellm` (env / compose changes) |
| `lc shell` | Shell inside the container |
| `lc curl <path>` | Authenticated GET against the proxy |

**`reload` vs `recreate` is load-bearing.** `docker restart` keeps the env vars baked at create time — changes to `.env` will not take effect. Use `recreate` whenever you change `.env` or `docker-compose.prod.yml`. Use `reload` for `deploy/litellm/config.yaml` edits only.

### Adding a model

1. `lc edit` — add a new `model_list` entry pointing at a real `vertex_ai/<id>`. Auto-restarts.
2. `lc models` — confirm the new alias appears.
3. Add a matching `{ "id": "<alias>", "name": "..." }` to `models.providers.litellm.models[]` in `~/.openclaw/openclaw.json`.
4. `openclaw models set litellm/<alias>` if you want it as the default.

---

## Pitfalls we hit (so you don't)

| Symptom | Cause | Fix |
|---|---|---|
| `Cannot convert undefined or null to object` in gaxios | OpenClaw's `gaxios` 7.1.4 dynamic-import crash on Node | Don't use `google-vertex` directly. Use Option B. |
| `dcp up` errors with `open /home/ubuntu/docker-compose.yml: no such file` | `dcp` uses relative `-f` paths | `cd ~/ea-openclaw` before running `dcp` |
| `Permission denied: '/run/secrets/gcp-sa.json'` inside LiteLLM | SA file is `chmod 600` on host, container UID ≠ host UID | `chmod 644 ~/ea-agent-sa.json` |
| Vertex 404 for `gemini-3.1-*` | Wrong model id — Vertex doesn't use `.x` minor versions | Use `gemini-3-pro-preview`, `gemini-2.5-pro`, etc. |
| Vertex 404 with `locations/us-central1` even after setting location to `global` | `lc reload` is `docker restart` — keeps existing env. Or wrong env-var key (`GOOGLE_CLOUD_LOCATION` ≠ `GCP_LOCATION`) | Use `GCP_LOCATION` in `.env`; run `lc recreate` after .env changes |
| OpenClaw error: `LLM request failed: network connection error` | `baseUrl` is `http://localhost:4000` — from inside the gateway container, localhost is the gateway, not litellm | Set `models.providers.litellm.baseUrl` to `http://litellm:4000` (docker service name) |
| Gateway error: `No API key found for provider "google-vertex"` while using LiteLLM | Stale `google-vertex/...` entry in `agents.defaults.models` triggers startup auth load | Remove the entry from `~/.openclaw/openclaw.json` |
| `openclaw config set models.providers.litellm.baseUrl ...` rejects with `models: expected array` | Schema requires the `models` array to exist before partial sets | Edit the JSON file directly with `nano` and write the whole block at once |
| LiteLLM `Waiting` for 90+ seconds on first `dcp up` | Healthcheck `start_period` is 30s, full timeout ~150s | Normal on cold start. `lc logs` to confirm it's loading. |
| `lc models` returns nothing | Container has no `curl` (we use `urllib` now) — older `lc` versions failed silently | `git pull` to get the updated shim |

---

## Files involved

```
docker-compose.prod.yml                    # adds litellm service, hardens gateway
deploy/litellm/config.yaml                 # Vertex Gemini model list (proxy side)
deploy/bin/lc                              # LiteLLM control shim
~/ea-openclaw/.env                         # LITELLM_MASTER_KEY, GCP_PROJECT_ID, GCP_LOCATION
~/ea-agent-sa.json                         # SA key (host, chmod 644, mounted into litellm)
~/.openclaw/openclaw.json                  # OpenClaw provider config — baseUrl, apiKey, models
~/.openclaw/agents/main/agent/             # per-agent state (auth profiles live here too)
```

The repo's `docker-compose.prod.yml` has both the LiteLLM service and the workspace-skills bind mount (see walkthrough 04 / `workspace-skills/`). After a `git pull` on EC2, only `.env` and `~/.openclaw/openclaw.json` are host-specific.
