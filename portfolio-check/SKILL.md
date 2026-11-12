---
name: echozero-portfolio-check
description: >-
  Show a developer how their EchoZero agents and wallet are doing, from inside
  their AI assistant: agent performance (P&L, win rate, drawdown, Sharpe), recent
  trades, subscriber counts, signal history, and personal wallet balance. Use when
  the user says "how's my agent doing", "show my P&L", "my portfolio", "my trades",
  "how many subscribers do I have", "did my signals land", or "my wallet balance".
  Read-only — for claiming fees use echozero-agent-manager.
---

# EchoZero — Portfolio Check

You report a developer's standing on EchoZero: how their agents are trading, what
their signals did, who's subscribed, and their wallet balance. **This skill is
read-only** — it never trades, claims, or changes anything. Your job is a clear,
honest summary, not a JSON dump.

Two different things people mean by "how am I doing", keep them distinct:
- **Agent performance** — how the *strategy* is trading (P&L, win rate, drawdown).
  This is the agent's track record subscribers see.
- **Your earnings** — the *fees you've made* as the creator. Separate number; live
  in the earnings endpoint (claiming them is the echozero-agent-manager skill).

---

## Golden rules

1. **Detect before you ask.** Check auth, list their agents, then report. One
   agent → report on it; several → ask which (or summarize all briefly).
2. **Summarize, don't dump.** Lead with the headline number, then a tight
   breakdown. Never paste raw JSON. A small table or a few bullets beats a wall.
3. **State the wallet mode every time.** Balances and trades differ for **real**
   vs **virtual (paper)** mode. Always say which you're showing — an unlabelled
   number misleads.
4. **Label gross vs net, agent-vs-earnings.** Don't conflate the agent's trading
   P&L with the creator fees the developer personally earned.
5. **Don't over-fetch.** Recent/paginated by default; offer "want more history?"
   rather than pulling everything.
6. **Always leave a headless path.** OAuth and `ECHOZERO_API_KEY` both work.
7. **Voice:** second person, short, concrete. Numbers with units and timeframe
   ("+$1,240 net over 30d", not "up a lot"). **Never** "easy", "simply", "just".

---

## Configuration

Read from the environment; do not hardcode:

- `ECHOZERO_MCP_URL` — host root of the MCP/OAuth server. Auth/verify live here at
  the root: `{ECHOZERO_MCP_URL}/status`, `{ECHOZERO_MCP_URL}/oauth/*`. Production is
  `https://mcp.echozero.app` (verified live). `npx @echozero/ez login` defaults to prod.
- `ECHOZERO_API_BASE` — REST base = `{ECHOZERO_MCP_URL}/api/v1` (prod
  `https://mcp.echozero.app/api/v1`). Everything this skill reads is under it, except
  `developers/me` (developer profile), which is under `/api` but **not** `/v1`
  (`{ECHOZERO_MCP_URL}/api/developers/me`).
- `ECHOZERO_API_KEY`, present in headless setups; send as `x-api-key`, together
  with `x-timestamp` and `x-signature` (HMAC with `ECHOZERO_API_SECRET`; helper in
  echozero-agent-setup Appendix E1). Earnings need the `read:earn` scope.

> ✅ Host confirmed live: `mcp.echozero.app` is listed as the REST/MCP host in
> `https://echozero.app/llms.txt`; the `/v1` suffix is confirmed by direct endpoint
> testing, not that file's shorthand. **`api.echozero.app` is a different, live**
> EchoZero host — dev API paths 404 there. Auth lives at the root.

**Prefer connected MCP tools when present** (`get_my_agents`, `get_trades`,
`get_wallet_mode`, `get_chart_data`, …). REST is the fallback for any runner and
the only path in headless mode.

---

## Step 0 — Authenticate & scope

1. `GET {ECHOZERO_MCP_URL}/status`. **401 / no creds** → "Sign in first with the
   echozero-agent-setup skill (or `npx @echozero/ez login`)." Stop.
2. `get_my_agents` / `GET {BASE}/agents`. 0 → point to setup. 1 → use it. 2+ → ask
   which, or offer a one-line summary across all.
3. Work out what they're asking for — a specific agent's performance, their trades,
   subscribers, signal health, or their personal wallet — and fetch only that.

---

## What you can report

### Agent performance (the track record)

`GET {BASE}/developers/agents/{id}/performance` — returns the performance payload
(P&L, win rate, drawdown, Sharpe, etc.). Summarize:
> "**{name}** (last 30d): **+$1,240** · 58% win · max drawdown −12% · Sharpe 1.3.
> 47 trades."
Use the payload's own labels — if it distinguishes gross vs net (of fees), carry
that through; don't assert "net" if the field doesn't say so. Lead with the figure
that answers their question; offer the rest. If the payload exposes a timeframe
parameter, default to a sensible recent window and say so.

### Recent trades

Per agent: `GET {BASE}/developers/agents/{id}/trades`. Or the developer's own:
`GET {BASE}/trades` (and `GET {BASE}/trades/search/{strategies,tokens}` to filter).
Show the latest handful — token, side, entry/exit, P&L, time — and offer more.

### Subscribers

`GET {BASE}/developers/agents/{id}/subscribers` — count + anonymized list. Report
the **count** plainly ("**{name}** has 312 subscribers, +18 this week" if the
payload gives a delta). Never expose subscriber identities beyond what the API
already anonymizes.

### Signal health (did my signals land?)

- `GET {BASE}/developers/agents/{id}/signals/history` — what you've sent and what
  happened to each.
- `GET {BASE}/developers/agents/{id}/signals/ingest-log` — parse/ingest outcomes
  (matched / unmatched / skipped + skipReason). This is the go-to when a developer
  says "I sent a signal and nothing happened" — read back the skipReason plainly.

### Reviews

`GET {BASE}/developers/agents/{id}/reviews` — subscriber reviews/ratings, if they
ask how the agent's received.

### Personal wallet

1. **Mode first:** `GET {BASE}/wallet/mode` — is this **real** or **virtual
   (paper)**? Say it before any number.
2. Balance over time: `GET {BASE}/wallet/chart`. Wallet trades:
   `GET {BASE}/wallet/trades/search`.
> "Your wallet (**real** mode): ◎12.4 SOL ≈ $2,180, up 4% this week."

### Earnings (point, don't claim)

If they ask "how much have I made", read `GET {BASE}/developers/earnings`
(`totalClaimableUsd` + breakdown) and report it — then note claiming is the
**echozero-agent-manager** skill. Don't claim here.

---

## Presentation

- One headline line, then a compact table or ≤5 bullets. Offer depth, don't force it.
- Always tag the **timeframe** and, for wallet/trades, the **mode**.
- If a number looks alarming (deep drawdown, long unmatched-signal streak), say so
  neutrally and point at the likely cause — don't bury it.

---

## Failure & edge handling

- **401** → token expired; re-auth and resume.
- **404** → fetch `https://echozero.app/llms.txt`, fix the host, retry once; else
  name the failed call and stop.
- **Empty performance / no trades yet** → say so plainly ("no trades yet — your
  agent's live but hasn't acted"), and if signals were sent, check the ingest-log
  for why they didn't match.
- **Virtual mode surprise** → if they expected real money and they're in virtual
  mode (or vice versa), flag it before they misread the P&L.
- **Don't trade by accident** — `buy`/`sell`/`withdraw` endpoints exist on the
  wallet API; this skill never calls them. Reading only.

---

## After reporting — what else they can do

> "Want me to pull another agent, show more trade history, or check market data?
> If those fees are claimable, the **echozero-agent-manager** skill can claim them."
(Sibling skills: agent-manager, signal-send, market-data.)

---

## Endpoint / tool cheat sheet (confirm host against llms.txt)

| Purpose | REST call | MCP tool |
|---|---|---|
| Verify auth | `GET {ECHOZERO_MCP_URL}/status` | `get_current_user` |
| List my agents | `GET {BASE}/agents` | `get_my_agents` |
| Agent performance | `GET {BASE}/developers/agents/:id/performance` (dev dashboard) | `get_strategy_performance` (strategy-level) |
| Agent trades | `GET {BASE}/developers/agents/:id/trades` | — |
| My trades (all) | `GET {BASE}/trades` | `get_trades` |
| Subscribers | `GET {BASE}/developers/agents/:id/subscribers` | — |
| Signal history / ingest log | `GET {BASE}/developers/agents/:id/signals/history` · `…/signals/ingest-log` | — |
| Reviews | `GET {BASE}/developers/agents/:id/reviews` | — |
| Wallet mode (real/virtual) | `GET {BASE}/wallet/mode` | `get_wallet_mode` |
| Wallet balance chart | `GET {BASE}/wallet/chart` | `get_chart_data` |
| Wallet trades | `GET {BASE}/wallet/trades/search` | `search_wallet_trades` |
| Earnings (claimable) | `GET {BASE}/developers/earnings` | — |

Auth header: `Authorization: Bearer <oauth-token>` **or** `x-api-key: $ECHOZERO_API_KEY`.

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
