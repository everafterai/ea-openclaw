# openclaw.json snapshot

The live gateway config — `/home/ubuntu/.openclaw/openclaw.json` on EC2 — is
hand-edited over time with RBAC tiers, agent definitions, channel prompts,
bindings, model/provider routing, and plugin config (Slack workspace IDs,
SearXNG base URL, LiteLLM endpoints, etc.). None of that is derivable from
source. To keep a versioned record without committing secrets, we maintain a
**redacted snapshot** in this directory.

This is not the runtime file. The OpenClaw container reads
`/home/ubuntu/.openclaw/openclaw.json` directly (via bind-mount). Editing the
snapshot in this repo does nothing on its own — it's a recovery artifact.

## Files

| File | Purpose |
|---|---|
| [openclaw.snapshot.json](openclaw.snapshot.json) | Latest redacted copy of the live config. Commit-friendly: every `*token`/`*secret`/`*apiKey`/`*cookie`/`*webhook` string is replaced with the literal `"$REDACTED"`. |

## Refreshing the snapshot

Run on EC2 after meaningful edits to `~/.openclaw/openclaw.json` (new admins,
agent tier changes, model routing, plugin wiring, channel additions, ...):

```bash
cd ~/ea-openclaw
bai snapshot                     # writes deploy/openclaw/openclaw.snapshot.json
git diff deploy/openclaw/        # review what drifted
git add deploy/openclaw/openclaw.snapshot.json
git commit -m "snapshot: <one-line reason>"
git push
```

Override the output path with `bai snapshot <path>` or `BAI_SNAPSHOT_PATH=...`.
See [deploy/bin/bai](../bin/bai) for the full command set.

## What gets redacted

Any string value whose enclosing key (case-insensitive) ends in:

- `token` &nbsp;— Slack `botToken`, `appToken`, ...
- `secret` &nbsp;— signing secrets, owner display secret, ...
- `password`
- `apikey`, `api_key` &nbsp;— LiteLLM master key, provider API keys, ...
- `cookie`
- `webhook`

`SecretRef` objects like `{"source":"env","id":"VAR_NAME"}` are intentionally
left intact — they document where the real value lives without leaking it.

Slack user IDs, channel IDs, agent prompts, model lists, and binding rules are
**not** secrets and stay in the snapshot.

## Rebuilding the live config from the snapshot

Disaster-recovery flow if `/home/ubuntu/.openclaw/openclaw.json` is lost:

1. Copy this snapshot to the live path:
   ```bash
   cp deploy/openclaw/openclaw.snapshot.json /home/ubuntu/.openclaw/openclaw.json
   ```
2. Replace each `"$REDACTED"` with the real value. Where to find them:

   | Field path | Source |
   |---|---|
   | `channels.slack.botToken` | Slack app config → OAuth & Permissions → Bot User OAuth Token (`xoxb-...`) |
   | `channels.slack.appToken` | Slack app config → Basic Information → App-Level Tokens (`xapp-...`, scope `connections:write`) |
   | `models.providers.litellm.apiKey` | EC2 `.env` → `LITELLM_MASTER_KEY` |
   | other provider `apiKey` fields | the upstream provider's console |

3. Validate and reload:
   ```bash
   bai validate
   bai reload          # full restart — needed when agents are reconstituted
   ```

The Slack tokens cannot be retrieved from Slack after first issue; if they're
lost, regenerate from the Slack app admin console and update both the snapshot
on next refresh and any other places they're stored.

## Notes

- The snapshot is for humans + disaster recovery. Don't let any runtime code
  read from it — `~/.openclaw/openclaw.json` is the single source of truth.
- Don't bypass redaction by editing the snapshot to include real secrets.
- The `meta.lastTouchedAt` / `wizard.lastRunAt` timestamps will drift on every
  snapshot. That's fine — `git diff` still highlights the substantive changes.
- For the wider deployment shape (compose, EC2 layout, `bai` / `dcp` / `lc`
  shims), see the root [CLAUDE.md](../../CLAUDE.md).
