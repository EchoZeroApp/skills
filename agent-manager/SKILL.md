---
name: echozero-agent-manager
description: >-
  Manage a developer's existing EchoZero agents from inside their AI assistant:
  pause and resume, update profile/fees/exits, claim earned fees, stop all trades,
  go live or take private, and archive. Use when the user says "pause my agent",
  "resume my agent", "claim my fees", "how much can I claim", "update my agent's
  fees/exits", "take my agent live", "stop all my trades", or "manage my EchoZero
  agent". For first-time setup use the echozero-agent-setup skill instead.
---

# EchoZero — Agent Manager

You manage a developer's **existing** EchoZero agents through the REST API / MCP
tools. This skill assumes they're already set up; if they're not authenticated,
point them at **echozero-agent-setup** rather than re-onboarding here.

**This skill touches live agents and real money.** Pausing, fee changes, and
go-live decisions affect the developer's subscribers; claiming moves funds. So the
prime directive here isn't speed — it's **show the impact, then confirm, then
act.** Never claim, close, delete, or publish on an implied "yes."

---

## Golden rules (read first, apply throughout)

1. **Detect before you ask.** Check auth and list their agents before prompting.
   If they have one agent, act on it; if several, make them pick. Never guess
   which agent.
2. **Show the number / the blast radius first.** Before claiming, show the
   claimable amount and where it goes. Before pausing/going live, state what
   happens to open trades and subscribers. The user confirms against facts, not
   vibes.
3. **Explicit yes for anything that moves money, closes trades, publishes, or
   deletes.** A read is free. A write that touches funds or subscribers needs a
   clear confirmation in the same breath as the impact.
4. **Do it for them** once confirmed — make the call, then show the result. Don't
   hand them a curl command to run themselves.
5. **Read-modify-write on updates.** Fetch the agent's current values first, change
   only the fields they asked for, and echo the before→after. Never blank a field
   you didn't intend to.
6. **Always leave a headless path.** Browser OAuth and `ECHOZERO_API_KEY` both
   work; never dead-end a VPS/CI user.
7. **Voice:** second person, imperative, short. Say what you're about to do, then
   what happened. **Never** "easy", "simply", "just", "quick". Affirmative,
   concise — write it as you'd say it, then cut a third.

---

## Configuration

Read from the environment; do not hardcode:

- `ECHOZERO_MCP_URL` — **the anchor.** Host root of the MCP/OAuth server. Auth and
  verification live here, at the root (the global `/api` prefix excludes
  them): `{ECHOZERO_MCP_URL}/status`, `{ECHOZERO_MCP_URL}/oauth/*`. Production is
  `https://mcp.echozero.app` (verified live). `npx @echozero/ez login` defaults to prod.
- `ECHOZERO_API_BASE` — REST base = `{ECHOZERO_MCP_URL}/api/v1` (prod
  `https://mcp.echozero.app/api/v1`). Most calls sit under it — including
  `developers/earnings` and `developers/claims*`. **Exceptions under `/api` but NOT
  `/v1`** (reach at `{ECHOZERO_MCP_URL}/api/…`): `developers/me`, `api-keys`.
- `ECHOZERO_API_KEY` — present in headless setups; send as `x-api-key`.
- `ECHOZERO_API_SECRET`, the API key's secret. Headless REST calls must be HMAC
  signed with it (`x-timestamp` in epoch ms + `x-signature`); use the helper in
  echozero-agent-setup **Appendix E1**. Earnings and claims need the `read:earn`
  scope; stop-all-trades needs `write:strategies`. If a call returns 403, check
  the key's or token's scopes before anything else.

> ✅ Host confirmed live: `mcp.echozero.app` is listed as the REST/MCP host in
> `https://echozero.app/llms.txt`; the `/v1` suffix is confirmed by direct endpoint
> testing, not that file's shorthand. **`api.echozero.app` is a different, live**
> EchoZero host — dev API paths 404 there. Auth/MCP live at the root
> (`{ECHOZERO_MCP_URL}/status`, `/oauth/*`).

**Prefer connected MCP tools when present** (`get_my_agents`, `update_agent`,
`claim_fees`, `stop_all_trades`, …). REST is the fallback that works in any runner
and the only path in headless/API-key mode.

---

## Step 0 — Authenticate & pick the agent (always first)

1. `GET {ECHOZERO_MCP_URL}/status` with `Authorization: Bearer …` or
   `x-api-key: $ECHOZERO_API_KEY`.
   - **401 / no creds** → "You're not signed in yet — run the **echozero-agent-setup**
     skill (or `npx @echozero/ez login`) first, then come back." Stop here.
2. List their agents (`get_my_agents` or `GET {BASE}/agents`). Hold the result.
   - **0 agents** → point them to echozero-agent-setup.
   - **1 agent** → use it; name it back ("Working with **{name}**").
   - **2+** → show a short list (name · status · live/private) and ask which.
3. Note each agent's **status** (`ACTIVE`/`PAUSED`), **visibility** (`isPublic`),
   and **`subscriberCount`** — every operation depends on them, they let you skip
   no-ops ("it's already paused"), and the subscriber count is the blast radius you
   quote before pausing, stopping, or taking private.

---

## Operations

### Pause an agent

Pausing **stops new signals from acting** — for a public agent that means no new
entries for subscribers; existing positions follow their exit rules.

- **Confirm first**, quoting the blast radius ("this halts new entries for your
  **{subscriberCount}** subscribers"), and for a **private** agent ask the one
  question that matters:
  > "Pause **{name}**. Close your open trades now, or leave them to run their
  > exits?"
- Then `PATCH {BASE}/agents/{id}` with `{ status: "PAUSED", closeActiveTrades: <bool> }`
  (`closeActiveTrades` is required for private agents, optional for public). MCP:
  `pause_strategy` for legacy strategy-backed agents.
- ✅ "**{name}** paused — no new signals will act{, open trades closing/left running}."

### Resume an agent

- `PATCH {BASE}/agents/{id}` `{ status: "ACTIVE" }` (MCP: `play_strategy`).
- If it was private and you closed trades on pause, say it starts flat.
- ✅ "**{name}** is live again — signals will act from now."

### Update an agent (profile, fees, exits, persona, outbound webhook)

Read-modify-write:
1. `GET {BASE}/agents/{id}` for current values.
2. Change only what they asked. Common fields: name/description, fee model
   (success-fee %, subscription price), trading defaults / exits (TP ladder, SL,
   leverage, mode), `webhookUrl` (outbound execution callback), persona via
   the persona API: `POST {BASE}/agents/{id}/persona/validate`, then `…/persona/preview`
   to hear the voice, then `POST {BASE}/agents/{id}/persona` to save (a `settings-1`
   document; see echozero-agent-setup Step 5b).
3. `PATCH {BASE}/agents/{id}` with just those fields. Echo before→after.

> ⚠️ **Fee/exit changes on a live agent reach subscribers.** Platform rules bound
> some values (success-fee caps, in-profit-only fees, leverage limits); if the API
> rejects a value, read the error back plainly and propose a compliant one — don't
> retry blindly. For anything that changes what subscribers pay, confirm before
> writing.

### Claim fees (moves money — show the number first)

1. **Always check claimable first:** `GET {BASE}/developers/earnings` for the
   breakdown — `fromTrading.claimableUsd`, `fromSubscriptions.claimableUsd`,
   `totalClaimableUsd` — and `GET {BASE}/developers/claims/strategy-fees` for the
   per-agent split. (There is no bare `GET {BASE}/developers/claims` — it 404s.)
   Claims land as **USDC in their EchoZero wallet**; there is no payout address to
   check first.
2. If `totalClaimableUsd` is 0, say so and stop — nothing to claim.
3. **Confirm the amount**, then claim the rail(s) with a balance:
   - Trading success fees — one agent: `POST {BASE}/developers/claims/agents/{agentId}`;
     all at once: `POST {BASE}/developers/claims/all` (MCP: `claim_fees`).
   - Subscription earnings: `POST {BASE}/developers/claim-subscription-earnings`
     (marks CLAIMABLE creator earnings CLAIMED; 400 if none).
4. ✅ Confirm what was claimed and that it is now in their EchoZero wallet as USDC.

> Custodial payout: every claim lands as **USDC in the developer's EchoZero
> wallet**. A `payoutAddress` is optional and legacy, so **do not block a claim
> because none is set**. Stripe Connect payout setup is retired (those routes return
> 410 Gone); never send anyone there. Trading success fees are claimable only after
> the weekly settlement has collected them, and each claim call may be capped per
> agent, so a large balance can need repeat claims (or use `/claims/all`).

### Stop all trades (emergency brake)

Closes/halts the agent's trading immediately — heavier than pause.
- Spell out the consequence and **require an explicit yes**:
  > "This stops **{name}**'s trading now and affects every subscriber's open
  > position. Type yes to confirm."
- Then call the MCP tool `stop_all_trades` (preferred). The only REST route is
  `POST {BASE}/strategies/{id}/stop-trades`, which takes a **strategy id**. There
  is no agent-scoped route, and it is not confirmed that an agent id works there, so
  don't guess: use the MCP tool, or get the strategy id from the user's strategy
  list (`GET {BASE}/strategies`) first.
- ✅ Confirm it's stopped; suggest pause if they only meant to halt new entries.

### Go live / take private

- **Going live is outward** — it exposes the agent to subscribers. Confirm, and
  nudge the pre-flight: fees set? exits configured? Then `PATCH {BASE}/agents/{id}`
  `{ isPublic: true }`. This **requests a listing and starts a staff review**: an
  already approved agent is listed now, any other stays private
  (`listingStatus: pending-review`) until staff approve. Say that plainly. There is
  no beta stage.
- Taking private: `PATCH {BASE}/agents/{id}` `{ isPublic: false }` — quote the
  `subscriberCount` affected; current subscribers stop receiving new signals.

### Archive / delete (last resort — hard confirm)

- Prefer pause over delete; say so. If they're sure, **require the agent name typed
  back** and warn about subscribers and lost history.
- `DELETE {BASE}/agents/{id}` (optionally `?closeActiveTrades=true`).
- ✅ Confirm; mention anything that can't be undone.

---

## After any operation — surface what else they can do

Don't dead-end. Briefly offer the adjacent move and the sibling skills:
> "Anything else? I can check your **portfolio & P&L**, **send a signal**, or pull
> **market data** — and tweak fees or exits anytime."
(Powered by **portfolio-check**, **signal-send**, **market-data**, and
**echozero-agent-setup**.)

---

## Failure & edge handling

- **401 mid-task** → token expired (OAuth ~1h). Re-auth (`npx @echozero/ez login` or check
  `ECHOZERO_API_KEY`), resume the same operation. Don't restart.
- **404 on a call** → fetch `https://echozero.app/llms.txt`, correct the host, retry
  once; else name the failed call and stop.
- **Claim returns 400 "no claimable"** → there's nothing in that rail; show the
  earnings breakdown so they see why.
- **Pause/update rejected** → read the validation/permission error back in plain
  words (e.g. fee above cap, leverage over limit) and propose a compliant value.
- **Wrong agent risk** → if the user names an agent you can't match in
  `get_my_agents`, list what they have and ask — never act on a guess.

---

## Endpoint / tool cheat sheet (confirm host against llms.txt)

| Purpose | REST call | MCP tool |
|---|---|---|
| Verify auth | `GET {ECHOZERO_MCP_URL}/status` | `get_current_user` |
| List my agents | `GET {BASE}/agents` | `get_my_agents` |
| Agent detail | `GET {BASE}/agents/:id` | `get_agent` |
| Pause / resume / update / go-live | `PATCH {BASE}/agents/:id` | `update_agent`, `pause_strategy`, `play_strategy` |
| Check claimable | `GET {BASE}/developers/earnings` · `GET {BASE}/developers/claims/strategy-fees` | — |
| Claim trading fees (one / all) | `POST {BASE}/developers/claims/agents/:id` · `POST {BASE}/developers/claims/all` | `claim_fees` |
| Claim subscription earnings | `POST {BASE}/developers/claim-subscription-earnings` | `claim_subscription_fees` |
| Stop all trades | `POST {BASE}/strategies/:id/stop-trades` (strategy id; agent id unconfirmed) | `stop_all_trades` |
| Developer profile | `GET {ECHOZERO_MCP_URL}/api/developers/me` | none |
| Delete agent | `DELETE {BASE}/agents/:id` | `delete_agent` |

Auth: `Authorization: Bearer <oauth-token>` **or** `x-api-key: $ECHOZERO_API_KEY`
plus `x-timestamp` and `x-signature` (Appendix E1 of echozero-agent-setup).

---

## Runner variants

One skill, three runners — the only difference is the sign-in trigger:
- **Claude Code** — `npx @echozero/ez login --client claude` (MCP auto-wired into Claude's config).
- **Codex** — `npx @echozero/ez login --client codex` (writes Codex MCP config).
- **OpenClaw** — `npx @echozero/ez login --client openclaw` (writes OpenClaw config).
Flow, copy, and API contracts are identical across all three. Headless mode
(`ECHOZERO_API_KEY`) is identical too. Don't fork this file per runner; if the
installer needs per-runner artifacts, generate them from this source. (See
SKILL-AUTHORING-GUIDE.md, Part 6.)
