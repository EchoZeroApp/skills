---
name: echozero-signal-send
description: >-
  Fire a one-off trade signal for a developer's EchoZero agent from inside their
  AI assistant: either a safe sandbox test, or a live signal that executes for
  subscribers. Use when the user says "send a signal", "send a buy/sell for my
  agent", "manually trigger a trade", "fire a test signal", or "push a signal to
  EchoZero". A live send moves real positions, so it always requires explicit
  confirmation.
---

# EchoZero: Signal Send

You send a single trade signal to a developer's agent. There are exactly two
destinations, and the difference is everything:

- **Sandbox** (`POST {ECHOZERO_MCP_URL}/api/sandbox/agents/{id}/signals/simulate`):
  a dry run that parses the signal and stores it in sandbox history. **No
  execution, no subscribers touched.** This is the default and needs no signing
  secret.
- **Live inbound** (`POST {ingress}` with HMAC): goes through the real execution
  pipeline and **acts on every subscriber's account.** This is the production path
  and needs the agent's signing secret.

**Treat a live send like spending money on behalf of other people.** Default to
sandbox. Only go live on an explicit, informed yes.

> There is **no** "authenticated REST send" shortcut. Live signals are HMAC-signed
> to the agent's ingress URL (the same contract a developer's own server uses). If
> you don't have the agent's `ECHOZERO_WEBHOOK_SECRET`, you cannot send live; offer
> sandbox or help them rotate the secret (echozero-agent-manager).

---

## Golden rules

1. **Detect before you ask.** Confirm auth, the target agent, and whether a live
   secret is available, before composing anything.
2. **Sandbox is the default.** Unless the user clearly says "live" or "for real",
   send to sandbox and show them the result first.
3. **Resolve and confirm the token.** If they name a token ("BONK"), resolve it to
   a mint (market-data / `GET {BASE}/tokens`) and **read the resolved mint + symbol
   back** before sending. A wrong mint is a real, unrecoverable loss.
4. **Live = explicit yes, with the blast radius.** Before any live send, state the
   agent, action, token, size, and **subscriber count**, then require a clear
   confirmation. No implied yes, ever.
5. **Validate the payload first.** Structured envelope (`eventType`) **or**
   natural-language `text`, never both. Numbers finite; size ≥ 0.
6. **Always send an `idempotencyKey`** on live sends (required for the structured
   envelope, and needed for `text` too) so a retry can't double-fire.
7. **Always leave a headless path.** OAuth and `ECHOZERO_API_KEY` both authenticate
   the sandbox call (an API key must be HMAC signed with `ECHOZERO_API_SECRET`, see
   echozero-agent-setup Appendix E1); live needs the agent's `ezw_` secret regardless
   of runner.
8. **Voice:** second person, short. State exactly what you're about to send, then
   what happened. **Never** "easy", "simply", "just".

---

## Configuration

Read from the environment; do not hardcode:

- `ECHOZERO_MCP_URL`: host root. Auth/verify at the root
  (`{ECHOZERO_MCP_URL}/status`). Production: `https://mcp.echozero.app`. Sandbox
  routes are `{ECHOZERO_MCP_URL}/api/sandbox/agents/{id}/…` (under `/api`, not `/v1`).
- `ECHOZERO_API_BASE`: REST base = `{ECHOZERO_MCP_URL}/api/v1` (prod
  `https://mcp.echozero.app/api/v1`). Used for token resolution (`/tokens`). Note:
  listing sent signals is `{ECHOZERO_MCP_URL}/api/developers/signals`, under `/api`
  but **not** `/v1`.
- `ECHOZERO_API_KEY`: headless auth; send as `x-api-key`.
- `ECHOZERO_API_SECRET`, the API key's secret; headless REST calls must be signed
  with it (`x-timestamp` + `x-signature`).
- `ECHOZERO_AGENT_ID`, `ECHOZERO_INGRESS_URL`, `ECHOZERO_WEBHOOK_SECRET`: the
  target agent and its **live** signing secret (set during echozero-agent-setup).
  Required for live sends only.

Docs index: `https://docs.echozero.app/llms.txt`. API spec:
`https://mcp.echozero.app/api/docs-json`.

---

## Step 0: Authenticate, pick agent, check capability

1. `GET {ECHOZERO_MCP_URL}/status`. **401** → "Sign in first (echozero-agent-setup
   or `npx @echozero/ez login`)." Stop.
2. Pick the agent (`get_my_agents` / `GET {BASE}/agents`; or `ECHOZERO_AGENT_ID`).
   Note its **status** (a `PAUSED` agent won't act on live signals) and
   **subscriberCount** (the blast radius for a live send).
3. Determine capability:
   - `ECHOZERO_WEBHOOK_SECRET` + `ECHOZERO_INGRESS_URL` present → live is possible.
   - Missing → live is off the table this session; say so and offer **sandbox**, or
     route them to echozero-agent-manager to rotate the secret.

---

## Composing the signal

$50 minimum to subscribe; $10 minimum per trade. Keep live sizes at or above the
per-trade minimum.

Build a structured envelope (preferred). A **buy** on Solana spot:
```json
{ "eventType": "buy", "idempotencyKey": "<stable, ≤128>",
  "chain": "solana", "instrument": { "tokenAddress": "<mint>" }, "symbol": "BONK",
  "side": "long", "tradeType": "spot", "amount": 25,
  "reasoning": "<shown in the public feed>",
  "triggers": { "slPrice": { "value": 0.00002, "targetType": "price" } } }
```
A **Hyperliquid** perp entry uses `"chain": "hyperliquid"`, `"instrument": { "coin": "ETH" }`,
`"tradeType": "perp"`, `"side": "long" | "short"` and `"leverageX"`.

- **Required with `eventType`:** `idempotencyKey` and `reasoning`. Entries
  (`buy`, `scale_in`) also need an instrument, `side`, `tradeType` and `amount`.
- **Lifecycle events** (`sell`, `partial_sell` (adds `sellSizePct`), `amend`
  (adds `changes`), `breakeven`, `cancel`) need `positionRef`, which is the entry's
  `idempotencyKey` or a trade id. Feed-only events: `trade_idea`,
  `position_update`, `trade_review`. Full field list:
  `https://docs.echozero.app/guides/signal-envelope`.
- **Token resolution:** if the user gives a symbol, resolve to a mint first
  (`GET {BASE}/tokens?query=…` / `get_trending_tokens`) and confirm symbol + mint.
- **Natural-language option:** `{ "text": "buy BONK, tight stop", "idempotencyKey": "<stable>" }`,
  which EchoZero parses. Use one style only (never `text` with `eventType`).
- The older `{ action, tokenAddress }` shape is legacy; only use it if the user
  already has code on it.
- Remind them `reasoning` is **public** (shows in the agent feed).

---

## Send

### Sandbox (default: safe, no execution)

`POST {ECHOZERO_MCP_URL}/api/sandbox/agents/{id}/signals/simulate`, authed by an
OAuth token, or by an API key signed like any REST call (no webhook HMAC). The body
is the raw signal message:
```json
{ "message": "Buy BONK $25, SL 10%" }
```
For a structured envelope, describe it in the message in plain words (the sandbox
parser takes text). Show the parsed result in plain words:
> "✅ Sandbox: EchoZero parsed your **buy of BONK**. Nothing executed and no
> subscribers were touched. Want to send this **live**?"

Sandbox history: `GET {ECHOZERO_MCP_URL}/api/sandbox/agents/{id}/signals`
(`data.items` plus `data.pagination`). To dry-run a trade instead of a parse:
`POST {ECHOZERO_MCP_URL}/api/sandbox/agents/{id}/trade` with
`{ "pair": "SOL/USDC", "amount": 100, "action": "BUY" }`.

Fallback only: the docs still list `POST {ECHOZERO_MCP_URL}/api/dev/trade-signals`
for portal demo signals, but it is not in the public API spec and the docs treat it
as unsupported. Use it only if the sandbox routes are unavailable.

### Live (executes: explicit confirmation required)

This executes for subscribers. Test in the sandbox and with virtual wallet mode
first, and keep the agent private until you have.

1. **Check readiness.** Has this signal shape passed the sandbox this session? If
   not, run it through the sandbox first. Refuse politely if the agent is `PAUSED`
   (resume first) or ineligible/in review.
2. **Confirm with the blast radius, then stop and wait:**
   > "Send **live**: **{name}** → **buy {SYMBOL}** ({mint}), size ${amount},
   > **{subscriberCount}** subscribers. This executes for subscribers. Test in the sandbox and with virtual wallet mode first, and keep
   > the agent private until you have. Type **yes** to send."
   Send only on a clear "yes" in reply to this exact message. Anything else
   (silence, "ok maybe", a changed size) means no send; re-confirm if any detail
   changes.
3. Sign and POST to the ingress URL (see Appendix; use the helper, don't hand-roll
   it): `POST {ECHOZERO_INGRESS_URL}` with `X-EZ-Timestamp` + `X-EZ-Signature`,
   including the `idempotencyKey`.
4. Show the outcome and, if matched, that it's executing. ✅

---

## Listing what you've sent

- All for this developer: `GET {ECHOZERO_MCP_URL}/api/developers/signals`.
- For one agent: `GET {ECHOZERO_MCP_URL}/api/developers/signals/agent/{agentId}`.
Use this to confirm a prior send or debug ("did my last signal go through?").

---

## Failure & edge handling

- **401 on the live send** → bad signature or clock skew (beyond ±5 minutes).
  Re-sign with a fresh timestamp; verify the secret matches the agent. Don't retry
  blindly.
- **404 Developer agent not found** → check the agent id.
- **400 validation** → usually `text` mixed with `eventType` fields, or an
  `eventType` payload missing `idempotencyKey` / `reasoning` / `positionRef`. Read
  it back plainly and fix.
- **No secret** → can't send live. Sandbox works; or rotate the secret via
  echozero-agent-manager.
- **Token didn't resolve** → never send on a guess. Show candidates and ask.

---

## After sending: what else

> "Want to check whether it executed (**portfolio-check** → signal history), send
> another, or pull market data first?"

---

## Appendix: HMAC signing (live sends)

This matches the docs' webhook security guide
(`https://docs.echozero.app/guides/webhook-security`).

Signature = lowercase hex `HMAC_SHA256(ezw_secret, "{unixSeconds}.{canonicalJson}")`,
with `X-EZ-Timestamp` = the same unix **seconds**. Requests outside ±5 minutes are
rejected with 401.

**Canonical JSON:** keys sorted alphabetically at every level, `undefined` omitted,
compact, numbers formatted the JavaScript way (`25`, not `25.0`). The server drops
unknown top-level fields before it verifies the signature, so only `text`, a
non-empty `idempotencyKey`, and the structured keys below are signed. Send only
documented fields.

Signed structured keys: `action`, `amount`, `caption`, `chain`, `changes`, `clientTimestamp`,
`confidence`, `context`, `currentPnlPct`, `entryPrice`, `entryZone`, `eventType`,
`exitPrice`, `exitReason`, `expiresAt`, `ideaId`, `instrument`, `leverageX`,
`metadata`, `orderType`, `pnlPct`, `positionRef`, `reasoning`, `relatedTradeId`,
`riskMgmt`, `sellSizePct`, `side`, `symbol`, `tokenAddress`, `tradeType`, `triggers`,
`urgency`, `version`.

Use the tested helper in **echozero-agent-setup, Appendix E2** (Node and Python), or
the SDK: `signInboundWebhook` from `@echozero/sdk` returns the two headers. Minimal
Node form:

```js
import { signInboundWebhook } from "@echozero/sdk";
const headers = await signInboundWebhook({ signingSecret: process.env.ECHOZERO_WEBHOOK_SECRET, body });
// POST JSON.stringify(body) to ECHOZERO_INGRESS_URL with Content-Type: application/json
// and the X-EZ-Timestamp / X-EZ-Signature headers from `headers`.
```

Published test vector: secret `secret`, timestamp `1710000000`, body
`{eventType:"buy", idempotencyKey:"test-1", reasoning:"test", tokenAddress:"So11111111111111111111111111111111111111112", amount:500}`
gives `be420f61d91e6b871481774c62f972f0aafa6f5f1a727ba1e4a32558784f77c3`.
More vectors: `https://docs.echozero.app/guides/signature-test-vectors`.

---

## Endpoint / tool cheat sheet (verify against the API spec)

| Purpose | Call |
|---|---|
| Verify auth | `GET {ECHOZERO_MCP_URL}/status` |
| Resolve a token | `GET {BASE}/tokens` · `GET {BASE}/tokens/trending` (MCP `get_trending_tokens`) |
| Sandbox signal (dry run) | `POST {ECHOZERO_MCP_URL}/api/sandbox/agents/:id/signals/simulate` |
| Sandbox trade (dry run) | `POST {ECHOZERO_MCP_URL}/api/sandbox/agents/:id/trade` |
| Sandbox history | `GET {ECHOZERO_MCP_URL}/api/sandbox/agents/:id/signals` |
| Live send (HMAC, executes) | `POST {ECHOZERO_INGRESS_URL}` |
| List sent signals | `GET {ECHOZERO_MCP_URL}/api/developers/signals` · `…/signals/agent/:id` |

Sandbox/list auth: `Authorization: Bearer <oauth-token>` **or** `x-api-key` plus
`x-timestamp` and `x-signature` (HMAC with `ECHOZERO_API_SECRET`, see
echozero-agent-setup Appendix E1). The live send uses **HMAC only** (the per-agent
secret), not those.

---

## Runner variants

One skill, three runners. The only difference is the sign-in trigger:
- **Claude Code**: `npx @echozero/ez login --client claude` (MCP auto-wired into Claude's config).
- **Codex**: `npx @echozero/ez login --client codex` (writes Codex MCP config).
- **OpenClaw**: `npx @echozero/ez login --client openclaw` (writes OpenClaw config).
Flow, copy, and API contracts are identical across all three (the live HMAC send is
runner-independent). Don't fork this file per runner; if the installer needs
per-runner artifacts, generate them from this source. (See CONTRIBUTING.md.)
