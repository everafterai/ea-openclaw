# SearXNG — self-hosted web search

Adds a [SearXNG](https://docs.searxng.org/) container to the EC2 deploy as a
key-free `web_search` provider for the OpenClaw gateway.

The bundled openclaw plugin at [extensions/searxng/](../../extensions/searxng/) talks to it
over the compose network; nothing here is forked-only code, just deployment
wiring.

## What changed

| Path | Purpose |
|---|---|
| [docker-compose.prod.yml](../../docker-compose.prod.yml) (`searxng:` service) | Internal-only `searxng/searxng:latest` container, port `8080` exposed on the compose network only |
| [deploy/searxng/settings.yml](settings.yml) | SearXNG config — `json` format enabled, limiter off, `secret_key` placeholder for entrypoint substitution |
| [.env.example](../../.env.example) | Adds `SEARXNG_SECRET` to the Deploy section |

No code change to the gateway — the openclaw `searxng` plugin is already in
this fork and activates lazily when `tools.web.search.provider = "searxng"`.

## Deploy steps (on EC2)

These run on the EC2 box, not on your laptop. Reminder: editing files in the
laptop clone does nothing until they reach EC2.

```bash
# 1. Pull the new compose service + settings.yml from this branch.
cd ~/ea-openclaw
git pull

# 2. Generate a SearXNG secret_key and put it in .env (one time).
echo "SEARXNG_SECRET=$(openssl rand -hex 32)" >> .env

# 3. Start the searxng container.
dcp up -d searxng

# 4. Confirm it is healthy (loopback only — never expose 8080 publicly).
dcp ps searxng
dcp logs --tail=50 searxng

# 5. From inside the gateway container, prove the JSON endpoint works.
dcp exec openclaw-gateway node -e \
  "fetch('http://searxng:8080/search?q=openclaw&format=json').then(r=>r.text()).then(t=>console.log(t.slice(0,200)))"
```

## Wire it into OpenClaw

Edit `/home/ubuntu/.openclaw/openclaw.json` on EC2 (the live config the gateway
reads via bind-mount — *not* the laptop's `~/.openclaw/openclaw.json`). Merge
the two blocks below into the top-level object:

```json5
{
  "tools": {
    "web": {
      "search": {
        "provider": "searxng"
      }
    }
  },
  "plugins": {
    "entries": {
      "searxng": {
        "config": {
          "webSearch": {
            "baseUrl": "http://searxng:8080",
            "categories": "general,news",
            "language": "en"
          }
        }
      }
    }
  }
}
```

Validate and hot-reload:

```bash
bai validate
bai reload --hot   # routing/config-only change — no full restart needed
```

The plugin's `activation.onStartup` is `false`, so SearXNG is activated lazily
on the first `web_search` call.

## Notes

- **Transport.** The plugin allows `http://` for trusted private-network hosts
  ([docs/tools/searxng-search.md](../../docs/tools/searxng-search.md)).
  `searxng` on the compose network is private, so `http://searxng:8080` is
  accepted. Public-internet endpoints must be `https://`.
- **JSON format is required.** Disabling `json` under `search.formats` in
  [settings.yml](settings.yml) breaks the plugin (it does not scrape HTML).
- **No Redis.** Limiter and ratelimit modules are off, so no Redis sidecar is
  needed. If this instance ever serves more than one tenant, re-enable limiter
  and add a Redis service.
- **Secret rotation.** The image substitutes `$SEARXNG_SECRET` into the
  placeholder `ultrasecretkey` at startup; rotating the secret needs
  `dcp up -d --force-recreate searxng` (env is baked at container create).
- **Provider precedence.** Per
  [docs/tools/searxng-search.md](../../docs/tools/searxng-search.md), SearXNG
  is last (order 200) in auto-detection. Any other web-search plugin with a
  configured API key (Brave, Tavily, Exa, ...) takes precedence unless
  `tools.web.search.provider` pins `"searxng"` explicitly. The snippet above
  pins it.
