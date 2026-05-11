# OpenClaw Slack RBAC — Deployment Design

**Status:** Proposed
**Author:** Shai Diamant (with Claude)
**Date:** 2026-05-10
**Target gateway:** AWS EC2 OpenClaw (everafterai/ea-openclaw fork), single workspace
**Scope:** First-pass production rollout for the EA org

## Goals

1. Run OpenClaw inside a single Slack workspace as a multi-purpose internal automation bot.
2. Enforce three coarse access tiers — **Admin**, **Operator**, **Viewer** — with per-tier agents and tool catalogs.
3. Make tier escalation structurally impossible from a low-tier surface (no thread-inheritance bypass, no DM-to-elevation drift).
4. Keep the rollout reviewable and reversible: one Slack app, one OpenClaw gateway, additive config.
5. Leave a clear path to per-channel `toolsBySender` overrides if a future need emerges.

## Non-goals

- Hostile multi-tenancy. OpenClaw is explicitly not a security boundary against adversarial users (per [docs.openclaw.ai/gateway/security](https://docs.openclaw.ai/gateway/security)). All users are assumed cooperative-internal.
- Per-user "least privilege" beyond the three tiers. Refining inside a tier is `toolsBySender` territory and is deferred.
- Direct mutation of shared production infrastructure (DBs, cloud resources) from the bot. Code execution stays in OpenClaw's sandbox/workspace, not loose on prod systems.
- IDP/SCIM-driven group sync. Not available on Slack Pro; tier membership is hand-maintained in `accessGroups`.

## Threat model

The two failure modes the design must defeat:

**T1. Cross-tier thread-inheritance leak** ([CVE-2026-41358](https://) class). When an admin starts a thread in a shared channel, OpenClaw's binding precedence pins that thread to the admin's agent for every subsequent reply, regardless of who replies. The agent's transcript already contains admin tool output; a non-admin reply could pull that into a response or trigger a tool with the admin's authority. Mitigation: **tier separation at the channel boundary** (Option A). A non-admin can't reply in a thread inside `#base-ai-admin` because they're not in the channel.

**T2. DM tier drift.** A non-admin DMing the bot must not land in the admin agent. Mitigation: **explicit per-user DM bindings** in routing. Admins are enumerated by Slack user ID with `peer: { kind: "direct", id: "U_ADMIN" }`; everyone else falls through to a wildcard DM binding pointing at the Viewer agent.

Out of scope for this design (defer or accept):

- **Information leakage from admin output already in a thread transcript** to a same-tier user who later joins. Channel-membership is the only gate; there's no per-message redaction in OpenClaw.
- **Prompt injection via Slack message bodies, channel topics, display names.** Mitigated by `unfurl_links: false` on bot posts and treating Slack content as untrusted, but no framework-level guarantee.
- **Bot invited to a channel it shouldn't see.** Mitigated by alerting (see Operational checklists) but Slack itself permits any channel member to `/invite @base-ai`.

## Architecture

```
Slack workspace (single)
├── Slack app: ea-agent (one app, one bot user)
├── Private channels (channel-as-tier):
│   ├── #base-ai-admin   — Admin tier members
│   ├── #base-ai-ops     — Operator tier members
│   └── #base-ai-ask     — Viewer tier members (general staff)
├── DMs to @base-ai:
│   ├── from admin user-IDs    → admin agent
│   └── from anyone else       → view agent
└── OpenClaw gateway (EC2)
    ├── Three agents in agents.list:
    │   ├── admin   sandbox: off,           tools: full      workspace: ~/.openclaw/workspace-admin
    │   ├── ops     sandbox: all/agent rw,  tools: workflow  workspace: ~/.openclaw/workspace-ops
    │   └── view    sandbox: all/agent ro,  tools: read-only workspace: ~/.openclaw/workspace-view (read-only mount)
    └── accessGroups: { admins, operators, viewers }
```

### Why this shape

- **Channel-as-tier** maps OpenClaw's strongest binding tier (`peer`) to the cheapest, most visible Slack-native ACL (private-channel membership). Workspace admins can audit "who is in `#base-ai-admin`" in the Slack UI without log access.
- **Three agents, three workspaces, three sandbox postures** keeps blast radius proportional to tier. The Admin agent gets `sandbox: off` for full host access; Ops gets `sandbox: all, scope: agent, workspaceAccess: rw` for its own scratch space; View gets `sandbox: all, scope: agent, workspaceAccess: ro` so even a prompt-injected Viewer agent can't write.
- **One Slack app** because at 2–5 admins / 10–30 users the install/audit overhead of two apps doesn't pay back.
- **No per-channel `toolsBySender`** in the v1. Adding it later is purely additive.

## Configuration

All paths assume the deployed OpenClaw gateway on EC2 (config under `/home/ubuntu/.openclaw/`).

**Where every block below lives.** All `accessGroups`, `agents`, `bindings`, `channels.slack`, and `commands` blocks below are top-level keys in a single config file: `/home/ubuntu/.openclaw/openclaw.json` on the EC2 host. That file is the one canonical source of truth — every Slack workspace ID (`T…`), channel ID (`C…`), and Slack user ID (`U…`/`W…`) appears there as a literal value. There is no environment-variable substitution or templating; the file is on a private EC2 host and is the right place for these IDs to live.

**Operating model.** Don't hand-edit the JSON. SSH into the EC2 and use the `bai` CLI ([deploy/bin/bai](../bin/bai)) to mutate the file safely (atomic writes, idempotent operations, structural validation, set-semantics on tier changes). After any mutation, `bai reload` validates the config and asks the gateway to pick it up (`openclaw config reload` when available, with a `docker compose restart openclaw-gateway` fallback).

**Optional: git snapshot for change history.** v1 doesn't ship structured audit logs. If you want a lightweight change history, periodically snapshot `~/.openclaw/openclaw.json` into this repo (e.g. under `deploy/openclaw.snapshot.json`) and commit. Strictly optional — `git log` over the snapshot then doubles as a "who changed what tier when" trail. Not wired in by default; flip it on if/when useful.

### `accessGroups` (in main openclaw config)

```json5
accessGroups: {
  admins: {
    type: "message.senders",
    members: {
      slack: [
        // 2-5 entries — Slack user IDs only, never usernames
        "U_ADMIN_1",
        "U_ADMIN_2",
      ],
    },
  },
  operators: {
    type: "message.senders",
    members: {
      slack: [
        // Engineers / on-call / power users (~5-10 entries)
        "U_OPS_1",
        "U_OPS_2",
      ],
    },
  },
  viewers: {
    type: "message.senders",
    members: {
      slack: [
        // Anyone in the org allowed to use the bot at all (~10-30 entries)
        "U_USER_1",
        // ...
      ],
    },
  },
}
```

**Identity rule.** Every member entry is a Slack user ID (`U…` or `W…`), never a username or display name. Slack itself recommends not storing usernames; they change, can collide, and `dangerouslyAllowNameMatching` exists only as a break-glass. Keep it `false`.

To resolve a user ID without admin tools: in Slack, click a user's profile → `…` menu → "Copy member ID".

### `agents.list`

```json5
agents: {
  // No top-level `default` key — OpenClaw's zod schema rejects it.
  // Mark the default agent with `default: true` on its list entry instead.
  list: [
    {
      id: "admin",
      workspace: "~/.openclaw/workspace-admin",  // tilde expands inside the container
      sandbox: { mode: "off" },
      tools: {
        // "group:memory" expands to {memory_search, memory_get} — bare "memory"
        // is a section label, not a tool id, and silently no-ops.
        allow: ["read", "write", "edit", "exec", "apply_patch", "browser", "sessions_send", "group:memory"],
        elevated: { enabled: true },
        // gateway, cron, plugin-management — owner-only, so they implicitly require ownerAllowFrom
      },
      // Use `systemPromptOverride`, not `systemPrompt` (zod rejects systemPrompt at this level).
      systemPromptOverride: "admin: full operator-tier agent for the EA org. Do not assume any sender is an admin without explicit verification of their Slack user ID against the admins accessGroup.",
    },
    {
      id: "ops",
      workspace: "~/.openclaw/workspace-ops",
      sandbox: { mode: "all", scope: "agent", workspaceAccess: "rw" },
      tools: {
        allow: ["read", "write", "edit", "browser", "sessions_send", "group:memory"],
        deny:  ["exec", "apply_patch", "gateway", "cron"],
      },
      systemPromptOverride: "ops: workflow-automation tier. You can draft, file, and update — but not execute shell commands, apply patches to the host, or modify gateway config.",
    },
    {
      id: "view",
      default: true,                              // safest fallback agent
      workspace: "~/.openclaw/workspace-view",
      sandbox: { mode: "all", scope: "agent", workspaceAccess: "ro" },
      tools: {
        allow: ["read", "browser", "group:memory"],
        deny:  ["write", "edit", "exec", "apply_patch", "sessions_send", "gateway", "cron"],
      },
      systemPromptOverride: "view: read-only research/lookup tier. Refuse any request that would mutate state and suggest opening a ticket via #base-ai-ops instead.",
    },
  ],
}
```

**Per-agent isolation rule.** Each agent has its own `agentDir` (`~/.openclaw/agents/<id>/agent/`) for OAuth and auth profiles. Never reuse `agentDir` between agents and never clone OAuth tokens — per the [multi-agent sandbox tools docs](https://docs.openclaw.ai/tools/multi-agent-sandbox-tools), this is an enforcement boundary, not a convention.

**Tool catalog rule.** Allowlist-based. New tools default to *unavailable*; explicitly add to the right tier's `allow` list when adopted.

### `bindings` (routing precedence)

Order matters; first match wins, ties broken by config order. List most-specific first.

```json5
bindings: [
  // ---- Per-admin DM routing (one entry per admin) ----
  { agentId: "admin", match: { channel: "slack", peer: { kind: "direct", id: "U_ADMIN_1" } } },
  { agentId: "admin", match: { channel: "slack", peer: { kind: "direct", id: "U_ADMIN_2" } } },

  // ---- Per-operator DM routing (one entry per operator) ----
  { agentId: "ops",   match: { channel: "slack", peer: { kind: "direct", id: "U_OPS_1" } } },
  { agentId: "ops",   match: { channel: "slack", peer: { kind: "direct", id: "U_OPS_2" } } },

  // ---- Channel-as-tier ----
  { agentId: "admin", match: { channel: "slack", peer: { kind: "channel", id: "C_ADMIN_CHAN" } } },
  { agentId: "ops",   match: { channel: "slack", peer: { kind: "channel", id: "C_OPS_CHAN" } } },
  { agentId: "view",  match: { channel: "slack", peer: { kind: "channel", id: "C_ASK_CHAN" } } },

  // ---- DM catch-all (any non-admin/non-ops DM) ----
  { agentId: "view",  match: { channel: "slack", peer: { kind: "direct", id: "*" } } },

  // ---- Slack workspace catch-all (only if message somehow escapes the above) ----
  { agentId: "view",  match: { channel: "slack", teamId: "T_WORKSPACE_ID" } },
]
```

**Why per-DM peers and not a per-sender match.** Slack bindings have no `senderId` match field — only `peer`/`teamId`/`accountId`. For DMs, `peer = the DM channel = effectively per-user`, so this resolves correctly. Cost: one extra binding line per admin/operator. At your scale, fine.

**Why channel bindings even though `users` allowlists also exist.** The binding decides *which agent answers*. The `users` allowlist (next section) decides *whether the message is processed at all*. Both are needed: bindings for routing, allowlists for authorization.

### `channels.slack` config

```json5
channels: {
  slack: {
    mode: "socket",                    // app-token + bot-token; no public webhook needed
    // SecretRef env-source form. The plain `{ secret: "X" }` shape is NOT valid;
    // zod requires `{ source, provider, id }`. A literal string also works.
    botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
    appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },

    // ---- DM access control ----
    dmPolicy: "allowlist",
    allowFrom: [
      "accessGroup:admins",
      "accessGroup:operators",
      "accessGroup:viewers",
    ],
    dm: {
      enabled: true,
      groupEnabled: false,             // deny multi-person DMs by default
    },

    // ---- Channel access control ----
    groupPolicy: "allowlist",
    dangerouslyAllowNameMatching: false,

    // ---- Threading ----
    replyToMode: "first",              // reply once at top, then thread
    thread: {
      requireExplicitMention: true,    // CVE-2026-41358 mitigation: thread reply only if @-mentioned
      historyScope: "thread",          // never load broader channel history
      inheritParent: false,
      initialHistoryLimit: 20,
    },

    // ---- Mentions ----
    requireMention: true,              // explicit @base-ai in channels

    // ---- Bot interaction ----
    allowBots: false,                  // never react to other bots (loop prevention)

    // ---- Per-channel overrides ----
    // Note: the per-channel `users:` allowlist is omitted intentionally.
    // The slack channel gate (extensions/slack/.../prepare.ts:471 →
    // src/channels/allowlist-match.ts:35) does NOT expand `accessGroup:`
    // references — it matches literal Slack user IDs only. Putting
    // `accessGroup:admins` here would silently reject everyone.
    // Tier separation is enforced two ways instead:
    //   1. Slack private-channel membership (Slack-side ACL)
    //   2. The global channels.slack.allowFrom above (this DOES expand
    //      accessGroup:) — gates whether the bot listens at all
    //   3. The channel binding (above) — routes to the right tier agent
    // If you ever want defense-in-depth here, populate `users:` with literal
    // Slack user IDs and keep them in sync with accessGroups manually.
    channels: {
      C_ADMIN_CHAN: {
        enabled: true,
        requireMention: true,
        systemPrompt: "You are in #base-ai-admin. Treat senders as admin tier. Audit every exec.",
      },
      C_OPS_CHAN: {
        enabled: true,
        requireMention: true,
        systemPrompt: "You are in #base-ai-ops. No exec, no apply_patch — refuse and suggest DMing an admin.",
      },
      C_ASK_CHAN: {
        enabled: true,
        requireMention: true,
        systemPrompt: "You are in #base-ai-ask. Read-only Q&A only. Refuse write requests and link to #base-ai-ops.",
      },
    },

    // ---- Native exec approvals (Slack-button UX) ----
    // Don't set `enabled` — the zod schema only accepts boolean, and the
    // default is "auto" (DM-first when approvers resolve). Including
    // `enabled: "auto"` will fail validation.
    execApprovals: {
      approvers: ["accessGroup:admins"],   // any admin can approve any exec
      target: "dm",                         // approval card DM'd to all admins
    },

    // ---- Reactions (visible feedback) ----
    ackReaction: "eyes",
    typingReaction: "hourglass_flowing_sand",
  },
}
```

**SecretRef shape.** `botToken` / `appToken` accept either a literal string or a SecretRef of the form `{ "source": "env" | "file" | "exec", "provider": "default", "id": "ENV_VAR_NAME" }`. The skeleton uses the env-var form; replace with literals only if you're not using OpenClaw's secrets keyring.

### `commands` (slash-command and owner gating)

```json5
commands: {
  ownerAllowFrom: [
    // Owner-only commands (/diagnostics, /export-trajectory, /config, /focus, gateway, cron)
    "slack:U_ADMIN_1",
    "slack:U_ADMIN_2",
  ],
  useAccessGroups: true,
  ownerDisplay: "hash",                // hash owner IDs in transcripts (avoid leakage)
  // Plain string only — zod rejects SecretRef objects here.
  ownerDisplaySecret: "REPLACE_WITH_OWNER_DISPLAY_HMAC_KEY",
  // Do NOT set commands.allowFrom — it would override channel allowlists/pairing entirely.
}
```

**Why no `commands.allowFrom`.** When set, it becomes the *exclusive* authorization source for slash commands and bypasses channel allowlists, accessGroups, and pairing. We want command authorization to inherit from channel membership + accessGroups, not be a separate world.

## Slack app setup

Single Slack app (`ea-agent`), Socket Mode (no public webhook required, works behind the EC2 firewall).

### Required bot scopes (minimal viable set)

```
app_mentions:read
assistant:write
channels:history
channels:read
chat:write
commands
groups:history
groups:read       # required to operate inside private channels
im:history
im:read
im:write
users:read        # required to resolve user IDs and validate accessGroup membership
usergroups:read   # optional but recommended; lets us reference Slack User Groups for cross-checks
```

### Required app-level token scope

```
connections:write   # Socket Mode
```

### Install + invite checklist

1. Create app at api.slack.com/apps from manifest (manifest derived from the scope list above).
2. Install to workspace; capture `xoxb-…` (bot token) and `xapp-…` (app token).
3. Store both in OpenClaw's gog keyring (never in plain `.env` once stable):
   ```bash
   openclaw secrets set SLACK_BOT_TOKEN
   openclaw secrets set SLACK_APP_TOKEN
   openclaw secrets set OWNER_DISPLAY_HMAC_KEY  # openssl rand -hex 32
   ```
4. Create the three private channels and invite `@base-ai`:
   - `#base-ai-admin` — invite admins + bot
   - `#base-ai-ops` — invite admins + operators + bot
   - `#base-ai-ask` — invite admins + operators + viewers + bot
5. Capture each channel ID (`Cxxx`) and write into `channels.slack.channels.<id>` config.
6. Subscribe to `member_joined_channel` events (auto with `channels:read`/`groups:read`) for the channel-invite alert (see Audit).

## Audit & observability

**v1: deferred.** No structured audit logging in v1, by user decision. The investigation surfaces we *do* have:

- **OpenClaw gateway logs** (whatever stdout/stderr lands in Docker logs on EC2) — best-effort, unstructured, may rotate.
- **Slack workspace audit logs** (admin-only, separate from the bot) — captures channel joins/leaves, app installs, message-edit events. Useful for "did the bot get invited to a channel it shouldn't be in" investigations.
- **Git history of `openclaw.json`** — every accessGroup membership change has a git-blame trail since the file lives in this repo. This is the closest thing to a "who had what permission when" record in v1.
- **Native Slack thread context** — when an exec is approved via the `execApprovals` DM card, the approval message itself stays in Slack as visible social proof.

When the org grows past the v1 scale (more than ~30 users, or any regulated-data exposure), add a follow-up spec for structured audit logging. The minimum field set to plan for: `timestamp`, `request_id`, `slack_team_id`, `slack_user_id`, `channel_id`, `channel_type`, `thread_ts`, `agent_id`, `tier_resolved`, and per-tool-call `tool_name` / `arguments` / `decision` / `approver_user_id` / `result_status`. Don't log message bodies plain — redact secrets and PII first.

## Operational checklists

These run through the `bai` CLI ([deploy/bin/bai](../bin/bai)), which manages `accessGroups`, per-user DM bindings, and `commands.ownerAllowFrom` atomically. The CLI runs **on the EC2 host** and mutates `~/.openclaw/openclaw.json` in place; `bai reload` then validates the config and asks the gateway to pick it up. Run `bai help` for the full command surface.

To run commands on the EC2, either SSH in interactively (`ssh -i keys/open-claw.pem ubuntu@<host>`) or use one-shot remote exec (`ssh ec2-claw 'bai admin add U_X'` once an SSH alias is configured).

### Adding a new admin

1. In Slack, get their member ID (profile → "…" → "Copy member ID").
2. `bai admin add <UID> [name]`
   - Adds to `accessGroups.admins.members.slack[]`
   - Adds per-user DM binding to `admin` (front of list, beats wildcard)
   - Adds `slack:<UID>` to `commands.ownerAllowFrom`
3. In Slack, invite to `#base-ai-admin`, `#base-ai-ops`, `#base-ai-ask`.
4. `bai reload`
5. Verify by DMing the bot from the new admin account: should reach `admin`.

### Adding a new operator

1. Get their Slack member ID.
2. `bai operator add <UID> [name]`
3. Invite to `#base-ai-ops` and `#base-ai-ask`.
4. `bai reload`

### Adding a new viewer

1. Get their Slack member ID.
2. `bai viewer add <UID> [name]`
3. Invite to `#base-ai-ask`.
4. `bai reload`

### Promoting or demoting a user (between tiers)

1. `bai <new-tier> add <UID> [name]` — set-semantics. The CLI removes the user from any other tier first, then sets the new tier (and removes/adds `ownerAllowFrom` automatically based on whether the new tier is admins).
2. In Slack, adjust channel memberships to match (the CLI prints reminders).
3. `bai reload`

### Removing a user entirely

1. `bai <current-tier> remove <UID>`
2. In Slack, kick from all `#base-ai-*` channels.
3. `bai reload`

### Inspecting state

- `bai tiers` — summary table (tier → agent → member count)
- `bai <tier> list` — list members of a tier
- `bai whois <UID>` — what tier(s) and bindings target this user
- `bai validate` — JSON syntax + structural sanity (catches placeholders, wrong-tier DM bindings, users in multiple tiers); `bai reload` runs this first and refuses if anything fails

### Adding a new tool to a tier

1. Decide which tier it belongs to (default to most restrictive that makes sense).
2. Add to that agent's `tools.allow` (and below tiers' `tools.deny` if it shouldn't fall through somehow).
3. If risky (mutating, exec, network), explicitly require it to flow through `execApprovals` by referencing it in the approval policy.
4. Document in this spec: append to the "Tool catalog by tier" section.
5. Reload, exercise once from a test user, confirm the audit log shows it correctly attributed.

### Adding a new tier-channel

1. Create private Slack channel, invite the right accessGroup members + `@base-ai`.
2. Add `channels.slack.channels.<new_id>` block with `enabled: true`, `requireMention: true`, and (optional) channel-specific `systemPrompt:`. Do **not** add a `users:` allowlist with `accessGroup:` refs — the slack channel gate doesn't expand them. Tier access is enforced by Slack channel membership + global `allowFrom`.
3. Add a `bindings` entry pointing to the right agent.
4. `bai reload`.

### Bot got invited to a channel it shouldn't be in

1. Alert fires on `member_joined_channel` for any channel ID not in the configured `channels.slack.channels.*` map.
2. Either:
   - Add the channel to the config (intentional), **or**
   - Have the bot leave: `chat.postMessage` apology + `conversations.leave`.
3. If the channel was private and contained sensitive history, treat it as a context-leak event and audit which messages the bot processed before leaving.

## Rollout plan

Five phases, each ending with verifiable proof before moving on.

**Phase 0 — Slack app and secrets** (½ day)
- Create the Slack app with the scope manifest above.
- Install to the workspace.
- Store tokens in OpenClaw secrets keyring.
- Verify: `openclaw channels slack status` reports `botTokenStatus: available`, `appTokenStatus: available`, `mode: socket`.

**Phase 1 — Single-tier Viewer-only sanity** (½ day)
- Add `accessGroups.viewers` with one user (yourself).
- Add `agents.list[view]` with the read-only tool set.
- Add the DM-wildcard binding to `view` and the workspace catch-all.
- Set `dmPolicy: "allowlist"` with `allowFrom: ["accessGroup:viewers"]`.
- Verify: DM the bot from your account, get a response. DM from any other account, get nothing (or a polite refusal).

**Phase 2 — Tier channels** (1 day)
- Create `#base-ai-ask` (Viewer), `#base-ai-ops` (Operator), `#base-ai-admin` (Admin).
- Add `accessGroups.operators` and `accessGroups.admins` with real members.
- Add `agents.list[ops]` and `agents.list[admin]`.
- Add the channel bindings and per-user DM bindings.
- Verify (table-test):
  | Test | Expected |
  |---|---|
  | Admin DMs bot | lands in admin |
  | Operator DMs bot | lands in ops |
  | Viewer DMs bot | lands in view |
  | Random staff DMs bot | rejected (not in any accessGroup) |
  | @-mention in #base-ai-admin from admin | admin responds |
  | @-mention in #base-ai-admin from non-admin | won't happen — non-admins can't be in the channel |
  | @-mention in #base-ai-ops from admin | **ops** responds (channel binding wins) — admin gets ops-tier tools there, by design |
  | @-mention in #base-ai-ask from operator | **view** responds — same rule |

**Phase 3 — Exec approvals** (½ day)
- Configure `channels.slack.execApprovals` with admin approvers.
- Trigger an exec from `admin` and approve via the DM card.
- Trigger an exec from `admin` and *deny*; verify the agent reports denial cleanly.

**Phase 4 — Audit + alerting** *(deferred for v1)*
- Skipped per current scope. Investigation in v1 relies on Docker stdout, Slack workspace audit, and git history of `openclaw.json` (see Audit section).
- Revisit once the org is past ~30 users or the bot starts touching regulated data.

**Phase 5 — Org rollout** (rolling)
- Add operators in batches of 2–3, reload config between batches.
- Add viewers in larger batches.
- Run a one-week soak with weekly review of: audit log volume, exec-approval requests, denied DMs, any `member_joined_channel` alerts that fired.

Reversal at any phase is `git revert` of the config change + config reload. No state migration required.

## Gotchas (learned from bootstrap)

- **Adding agents requires a full container restart, not `openclaw config reload`.** The in-process config reload picks up routing/binding/accessGroup changes for *existing* agents but doesn't re-initialize newly-added agents (or fully reset their workspace state). Use `bai reload` (default — `docker compose restart openclaw-gateway`) after any agents.list change. `bai reload --hot` is the faster in-process reload, but only safe when you haven't added or removed agents.
- **Just writing an agent to `openclaw.json` isn't enough — you also need `openclaw agents add <id>` (or `bai bootstrap`).** That command runs `ensureWorkspaceAndSessions`, which creates the 7 workspace bootstrap files (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md`) and the per-agent sessions directory. The UI uses those files to populate the "Files" tab and seems to hide agents missing them.
- **The UI dropdown shows agents the gateway has fully registered, not just configured.** After `bai migrate` + `bai bootstrap` + `bai reload`, the dropdown should expand to all four tiers. If only `main` appears, run `bai reload` again (full restart, not `--hot`).
- **Workspace paths must use `~/.openclaw/...` (tilde-expanded inside the container), not `/home/ubuntu/.openclaw/...` (host path).** The container's home is `/home/node`; the bind-mount makes them resolve to the same place when the path is tilde-form.
- **`commands.ownerDisplaySecret` is a plain string, not a SecretRef object.** The skeleton's `REPLACE_WITH_OWNER_DISPLAY_HMAC_KEY` is a literal placeholder — replace with `openssl rand -hex 32` output, or switch `ownerDisplay` to `"raw"` and drop the secret entirely (defensible for a small cooperative team where Slack IDs are already visible).
- **`channels.slack.execApprovals.enabled: "auto"` fails zod validation** despite the docs suggesting it. The schema only accepts boolean. Omit the key (default is auto-on when approvers resolve) or set to `true`.
- **Per-channel `users: ["accessGroup:admins"]` silently rejects everyone.** The slack channel gate doesn't expand `accessGroup:` refs — only literal Slack user IDs match.
- **`channels.slack.allowFrom: ["accessGroup:admins", ...]` also silently rejects everyone.** Same code path as the per-channel `users:` gate — `accessGroup:` refs are NOT expanded at runtime, despite the OpenClaw docs implying they are. With `dmPolicy: "allowlist"` and `allowFrom` containing only `accessGroup:` refs, the bot silently drops every DM. The fix is to populate `allowFrom` with the literal union of all tier member IDs. `bai` does this automatically on every `tier add/remove` and during `bai migrate`. **If you ever hand-edit `allowFrom` to add `accessGroup:` refs, DMs will break.** Keep literal IDs only.
- **Tier separation in shared channels is enforced by Slack channel membership + the global `channels.slack.allowFrom` (literal IDs) + bindings.** Not by per-channel `users:` arrays, which (as above) don't expand accessGroups.

## Open questions / decisions deferred

1. ~~**Bot user display name**~~ — resolved: `@base-ai`.
2. **Workspace ID (`T…`), channel IDs (`C…`), member IDs (`U…`)** — concrete values get captured during Phase 0–2 and written into `openclaw.json` as literals. No templating needed.
3. **Approval forwarding to other channels** (`approvals.exec.targets`) — defer until we have a use case. Native Slack DM approval covers Phase 3.
4. **Per-channel `toolsBySender` overrides** — explicitly deferred per the brainstorm. Add a follow-up spec when a concrete need emerges.
5. **IP-restricting the Slack bot token** — Slack supports it; recommended once the EC2 has a stable egress IP.
6. ~~**Audit log destination**~~ — deferred for v1 (no structured logging). Revisit per the Audit section's trigger conditions.

## References

- [docs.openclaw.ai/channels/slack](https://docs.openclaw.ai/channels/slack) — full Slack channel reference
- [docs.openclaw.ai/channels/channel-routing](https://docs.openclaw.ai/channels/channel-routing) — bindings precedence
- [docs.openclaw.ai/channels/pairing](https://docs.openclaw.ai/channels/pairing) — accessGroups example
- [docs.openclaw.ai/concepts/multi-agent](https://docs.openclaw.ai/concepts/multi-agent) — agent routing
- [docs.openclaw.ai/gateway/security](https://docs.openclaw.ai/gateway/security) — trust model and non-goals
- [docs.openclaw.ai/tools/multi-agent-sandbox-tools](https://docs.openclaw.ai/tools/multi-agent-sandbox-tools) — per-agent isolation
- [docs.openclaw.ai/tools/exec-approvals](https://docs.openclaw.ai/tools/exec-approvals) — approval flow
- [docs.openclaw.ai/tools/slash-commands](https://docs.openclaw.ai/tools/slash-commands) — `commands.allowFrom` / `ownerAllowFrom` semantics
