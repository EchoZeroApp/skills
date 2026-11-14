---
name: echozero-agent-setup
description: >-
  Onboard a developer onto EchoZero end-to-end, inside their AI assistant: sign
  them in (which also connects EchoZero's MCP tools), create their agent, fire a
  sandbox test signal so they see it work in the first minute, then optionally
  generate a strategy, give the agent a voice, and wire execution callbacks, before
  handing off to the portal to go live. Use when the user installs the EchoZero
  skill, says "set up EchoZero", "create an EchoZero agent", "connect my trading
  signals to EchoZero", or runs `npx skills add EchoZeroApp/skills` for the first time.
---

# EchoZero: Agent Setup

You are walking a developer through getting their first trading agent live on
EchoZero, **inside the chat they already use** (Claude Code / Codex / OpenClaw).
This skill IS the onboarding flow; there is no web wizard to fall back on. Your
job is to *do the setup for them*, not to read them a tutorial.

EchoZero is a **multi-venue** trading-agent platform: agents trade on **Solana
spot (Jupiter)** or **Hyperliquid perps**. Each user's funds sit in their own
wallet (powered by Privy), with one USDC balance. Each agent is locked to one venue
when it is created. Developers run their strategy code **off-platform** (their repo
/ VPS) and send **signals** to EchoZero over a webhook; EchoZero executes and
manages the trades for subscribers. $50 minimum to subscribe; $10 minimum per
trade. You are never writing the strategy's edge. You are wiring their signals to
EchoZero and proving the loop works.

**The prime directive: get them to a working sandbox signal in the first minute.**
Sign in, create the agent, fire a sandbox test, *then* offer the deeper setup.
Everything after the first ✅ is optional and pull-based.

---

## Golden rules (read first, apply throughout)

1. **Detect before you ask.** Check auth state and existing agents before
   prompting for anything. Never re-ask something you can look up. A returning
   user lands mid-flow, not at step 1.
2. **Fastest path to first success.** Don't make them build anything before they
   see a signal land in the sandbox. Defer code, voice, and config until *after*
   the test passes.
3. **Do it for them.** When a step is an API/tool call you can make, make it
   (after confirming, see rule 5). "I've created your agent" beats "here's how to
   create an agent."
4. **Sane defaults, offered as confirmations.** Propose the obvious answer so they
   can say "yes": "I'll make it **private** while you test, ok?" Don't make them
   author every field.
5. **Confirm before anything outward or hard to undo.** A *private draft* agent
   and *sandbox* signals are safe, so do them. **Going live, setting real fees,
   or sending a live (non-sandbox) signal needs an explicit yes.**
6. **Always leave a headless path.** Some users are on a VPS/CI with no browser.
   Whenever you offer browser OAuth, also state the `ECHOZERO_API_KEY`
   alternative. Never dead-end a user who can't open a browser.
7. **End every step on a visible win.** "✅ Signed in", "✅ Agent created",
   "✅ Sandbox signal parsed: EchoZero saw your buy." They need to feel the loop close.
8. **Surface what they unlocked.** Sign-in connects EchoZero's MCP tools; setup
   reveals sibling skills. Tell them what they can now *do*; don't leave it hidden.
9. **Voice:** second person, imperative, short. Say what to do and what
   happened. **Never** use "easy", "simply", "just", or "quick": they make a
   stuck user feel stupid. Affirmative over negative. Write like you'd say it out
   loud, then cut a third of the words.

---

## Configuration

Read these from the environment; do not hardcode:

- `ECHOZERO_MCP_URL`: **the anchor.** Host root of the MCP/OAuth server
  `npx @echozero/ez login` talks to. Auth and verification live **here, at the
  root, not under the REST base** (the global `/api` prefix excludes them):
  `{ECHOZERO_MCP_URL}/status`, `{ECHOZERO_MCP_URL}/oauth/*`, `{ECHOZERO_MCP_URL}/mcp`.
  Production: `https://mcp.echozero.app`. `npx @echozero/ez login` defaults to it.
- `ECHOZERO_API_BASE`: REST base = `{ECHOZERO_MCP_URL}/api/v1` (prod
  `https://mcp.echozero.app/api/v1`). Most calls sit under it: `/agents`, `/tokens/*`,
  `/ai/*`, `/wallet/*`, `/trades`, `/strategies/*`. **Several routes are under
  `/api` but NOT `/v1`**; reach them at `{ECHOZERO_MCP_URL}/api/…`:
  `sandbox/agents/:id/*` (sandbox), `developers/register`, `developers/me`,
  `developers/signals` (GET-only), `api-keys`, and `public/agent-signals/:id` (inbound).
- `ECHOZERO_API_KEY`, present in headless setups; send as `x-api-key`. Key ids
  look like `ez_live_…`.
- `ECHOZERO_API_SECRET`, the API key's secret, shown once when the key is created.
  **Headless calls must be HMAC signed with it** (see *Appendix E1*): every REST
  call that uses an API key carries `x-timestamp` and `x-signature`, and an
  unsigned call is rejected. OAuth sign-in does not need signing.
- `ECHOZERO_PORTAL_URL`, developer portal base (prod `https://devportal.echozero.app`).
  Used in Step 6 handoff links.

Docs index: `https://docs.echozero.app/llms.txt`. API spec:
`https://mcp.echozero.app/api/docs-json`.

**Prefer connected MCP tools when you have them.** After `npx @echozero/ez login`
(Step 1), this client is also connected to EchoZero's MCP server, so tools like
`get_trending_tokens`, `generate_strategy`, `create_agent`, and `get_my_agents`
are callable directly. Use them when present. The REST calls below are the
fallback that works in any runner (and the only path in headless/API-key mode).

---

## The flow

### Step 0: Orient (silent)

Before saying anything:

- `GET {ECHOZERO_MCP_URL}/status` with whatever auth exists (`Authorization:
  Bearer …` from a prior `npx @echozero/ez login`, or `x-api-key: $ECHOZERO_API_KEY`
  plus the signing headers from *Appendix E1*). Note `/status` is at the server
  root, not under the REST base.
  - **200** → authenticated. `/status` returns no name, so read it from
    `GET {BASE}/users/me` and greet them by it. Note `isDeveloper` and
    `developerStatus` from the `/status` body (Step 3 uses them). Skip to **Step 2**.
  - **401 / no creds** → **Step 1**.
- If authenticated, also `GET {BASE}/agents` (or call `get_my_agents`). Hold this
  result. It drives the **re-run guard** in Step 3 and lets a returning dev
  manage an existing agent instead of making a duplicate.

### Step 1: Sign in

Pick the path by environment; don't make the user choose blind.

- **Interactive (browser available):** tell them you'll sign them in, then run:
  > ```bash
  > npx @echozero/ez login --client claude --scopes "read:agents write:agents read:wallet read:trades read:earn read:strategies write:strategies write:ai"
  > ```
  > The extra scopes (`read:wallet read:trades` for the portfolio skill, `read:earn`
  > for earnings and claims, `read:strategies write:strategies` for stop-all-trades,
  > `write:ai` for the strategy generator in Step 5a) mean later steps work without
  > re-auth. This runs OAuth 2.1 + PKCE: it opens their browser to the EchoZero
  > consent screen, captures the code on a local callback (`:8765`), exchanges it
  > for a token, verifies via `/status`, and saves it to `~/.echozero/ez-login.json`.
  > You never handle the token. Wait for "EchoZero login complete", then re-check
  > `GET {ECHOZERO_MCP_URL}/status`.
- **Headless (VPS / CI / no browser):** have them create an API key (portal →
  > API Keys → Create; scopes `read:agents write:agents read:wallet read:trades
  > read:earn read:strategies write:strategies write:ai`). The portal shows the
  > key id and its **secret once**. Export both:
  > ```bash
  > export ECHOZERO_API_KEY="ez_live_…"      # the key id
  > export ECHOZERO_API_SECRET="…"           # the secret, shown once
  > export ECHOZERO_API_BASE="https://mcp.echozero.app/api/v1"
  > ```
  > Then use the signing helper in *Appendix E1* for every REST call. A bare
  > `x-api-key` header is not enough.

On success: **✅ "Signed in as {name}."** (name from `GET {BASE}/users/me`).

**Then announce what got unlocked (interactive `npx @echozero/ez login` path only):**
> "`npx @echozero/ez login` also connected EchoZero's tools to this assistant. Beyond
> setup, you can ask me to **check your portfolio**, **research a token**, **send a
> signal**, or **claim fees** anytime. First, let's get your agent working."

(Headless/API-key mode authenticates REST calls but does **not** auto-wire MCP,
so skip the MCP announcement there.)

If sign-in fails, say exactly what to try (re-run `npx @echozero/ez login`, or check
the key's scopes). Never leave them on a raw 401.

### Step 2: Understand the agent (one question)

> "What kind of agent are you shipping?
> **momentum**, **mean-reversion**, **signal-relay** (you already have signals and
> want to forward them), or **custom**?"

Note the answer (it steers Step 5). If they post calls in a **Telegram or
Discord group** rather than running their own bot or code, switch to the
**echozero-signal-group-setup** skill. Then ask the **venue**, because it is locked
once the agent exists:
> "Where will it trade, **Solana spot (Jupiter)** or **Hyperliquid perps**? You
> can't change this later."

Then capture the minimum to create, a **name** (**20 characters or fewer**) and a
one-line description, and default the rest out loud:
> "I'll set it up **private** and **fee-free** so you can test safely. You set the
> listing and fees in the portal before going live. Sound good?"

Keep this short. Don't collect strategy details, fees, or exits here; that's
Step 5 and the portal.

### Step 3: Register as a developer, then create the agent

**Register first if needed.** Webhook agents only receive signals for registered
developers, and earnings, claims and the sandbox routes need it too. Use
`isDeveloper` from the Step 0 `/status` body:
- **`isDeveloper: true`** → carry on.
- **`false`** → say so and offer to register: "You're not registered as an EchoZero
  developer yet. I'll do that now, it takes one call." Then
  `POST {ECHOZERO_MCP_URL}/api/developers/register` with `{ "name": "<a label for
  the developer API key>" }` (note: under `/api`, not `/api/v1`; `payoutAddress` is
  optional because claims land in the EchoZero wallet). The response returns a new
  developer API key (`rawKey`) and secret (`rawSecretKey`) **once**; have them save
  both. A **409** means they are already registered; carry on.

**Re-run guard:** if Step 0 found an existing agent, offer to reuse it
("You already have **{name}**. Continue with that, or create a new one?") rather
than silently making a duplicate.

Otherwise create it (safe: a private draft, so go ahead). Prefer the `create_agent`
tool if connected; else `POST {BASE}/agents` (signed if using an API key, with the
`x-ez-mcp-client` header the spec lists for this route; any short client label
works, for example `echozero-skills`. With the *Appendix E1* helper:
`ez("POST", "/api/v1/agents", body, { "x-ez-mcp-client": "echozero-skills" })`):
```json
{ "name": "<=20 chars", "description": "…", "isPublic": false,
  "signalSourceKind": "webhook", "agentVenue": "JUPITER" }
```
- `name` is **20 characters or fewer**.
- `agentVenue` is the answer from Step 2: `JUPITER` or `HYPERLIQUID`. It **cannot be
  changed after create**.
- For **`HYPERLIQUID`** add `"signalExecutionDefaults": { "marketType": "perpetual" }`
  in the same body. Always include it for Hyperliquid agents.

The response reveals the inbound webhook secret **exactly once**:
`inboundWebhookSigningSecretReveal` (`ezw_…`) and `inboundSignalsHttpUrl`. Capture
both and have them save it:
> "🔑 This is the **only** time EchoZero shows your signing secret. Save it now,
> in a password manager or deploy env, not your code:
> ```bash
> export ECHOZERO_AGENT_ID="{id}"
> export ECHOZERO_WEBHOOK_SECRET="ezw_…"
> export ECHOZERO_INGRESS_URL="{inboundSignalsHttpUrl}"
> ```
> Lost it? I can rotate it, but the old one stops working."

**Listing is a separate, later step.** `isPublic: true` asks to list the agent in
the marketplace and starts a **staff review**; until it is approved the agent stays
private (`listingStatus: pending-review`). A private agent can be tested in the
sandbox right away. Say: "✅ Agent **{name}** created and your webhook is set up.
It stays private until you ask for a listing."

### Step 4: Fire a sandbox test signal (the win)

This is the moment. Prove the loop in the sandbox. Sandbox routes are dry runs:
they never touch the trade engine or subscribers.

**Primary test (documented sandbox route):**
`POST {ECHOZERO_MCP_URL}/api/sandbox/agents/{id}/signals/simulate` (note: under
`/api`, not `/api/v1`), authed with their OAuth token, or with their API key signed
like any REST call (*Appendix E1*). No webhook HMAC. Body:
```json
{ "message": "Buy SOL at $145, TP $160, SL $138" }
```
To make it feel live, pull a real token first with `get_trending_tokens` (or
`GET {BASE}/tokens/trending`) and use its symbol in the message, for example
`"buy BONK $25, SL 10%"`. The simulation is stored in the agent's sandbox history
(`GET {ECHOZERO_MCP_URL}/api/sandbox/agents/{id}/signals`).

Other sandbox routes: `POST {ECHOZERO_MCP_URL}/api/sandbox/agents/{id}/trade` with
`{ "pair": "SOL/USDC", "amount": 100, "action": "BUY" }` simulates a trade.

**Fallback only:** the docs still list a portal demo-signal route,
`POST {ECHOZERO_MCP_URL}/api/dev/trade-signals`, but it is not in the public API
spec and the docs treat it as unsupported. Use it only if the sandbox routes are
unavailable, with `{ "developerAgentId": "{id}", "action": "buy", "tokenAddress":
"{mint}", "amount": 25 }`.

Show the parsed result in plain words:
> "✅ Sandbox signal parsed: EchoZero read your **buy** of {TOKEN} and would have
> acted on it. Nothing executed. The loop works."

If it doesn't parse, read back the validation error or skip reason in plain words
and fix it together.

**Want to test real execution without real money?** Switch to virtual wallet mode
(`POST {BASE}/wallet/mode` `{ "isVirtualMode": true }`, then
`POST {BASE}/wallet/virtual/add-funds` `{ "amountInUsdc": 1000 }`) before any
signed inbound send. See the sandbox guide:
`https://docs.echozero.app/guides/sandbox`.

**They've now succeeded. Everything below is optional: offer it, don't impose it.**

### Step 5: Expand (offer a menu, pull not push)

> "You're working. Want me to help with any of these, or skip to going live?
> • **Generate a starter strategy** in your repo
> • **Give your agent a voice** for its public feed
> • **Get notified when trades execute** (outbound webhook)"

**5a: Generate a starter strategy.** Use EchoZero's own generator: call
`generate_strategy` (connected MCP tool) or `POST {BASE}/ai/strategy/generate`
(scope `write:ai`), seeded with their Step 2 answer, so the stub is tailored, not
generic.
If the generator isn't reachable, fall back to the hand-written scaffold in
*Appendix A*. Either way, fill in their `ECHOZERO_AGENT_ID` / `ECHOZERO_INGRESS_URL`
/ `ECHOZERO_WEBHOOK_SECRET` and tell them where to drop their own logic.
- **For signal-relay users:** they already have signals, so skip the from-scratch
  stub. Give them the **adapter** in *Appendix D* that maps their existing signal
  shape onto EchoZero's payload and signs it. That's the only code they need.

Whatever code you hand over sends **live** signals. Tell them, every time:
> "This executes for subscribers. Test in the sandbox and with virtual wallet mode
> first, and keep the agent private until you have."

Never run the scaffold or adapter against the live ingress URL yourself without an
explicit yes from the user for that send (see echozero-signal-send).

**5b: Give the agent a voice (persona).** EchoZero auto-writes the agent's public
feed posts in a voice you define. If they want it, use the settings-based persona
API, in this order, and don't save before they like it:
1. **Pick a base voice.** `GET {BASE}/agents/persona-bases` lists them (FRIENDLY,
   CYNICAL, QUIRKY, DRY, COCKY, HYPE_NERD, DEGEN, CALM, BLANK). Collect a name, who
   the agent is, how it speaks, and a few example posts (write the ticker as `{token}`).
2. **Validate:** `POST {BASE}/agents/{id}/persona/validate` with a `settings-1`
   document, for example
   `{ "settings": { "version": "settings-1", "base": "DRY", "identity": { "name": "Dry", "who": "…", "voice": "…" }, "examples": { "byMoment": { "open": ["In. {token}. Fine."] } } } }`.
   Read any problems back in plain words.
3. **Preview:** `POST {BASE}/agents/{id}/persona/preview` with the same body. It
   returns sample posts (and `warnings`, such as too few examples) and saves nothing.
   Show the samples so they hear the voice.
4. **Save:** `POST {BASE}/agents/{id}/persona` with the same body, only after they
   approve. It applies to future posts; existing posts stay as they were. Earlier
   versions are listed at `…/persona/versions` and can be rolled back.

The older free-text body (`persona` + `voice` + `examples` strings) still works,
but then **both `persona` and `examples` are required**. Prefer `settings-1`. Keep
the whole step optional and skippable.

**5c: Outbound execution webhook.** This is how *they* get told when a trade
fires. If they have a server endpoint, set the agent's `webhookUrl`
(`PATCH {BASE}/agents/{id}`). EchoZero will POST `signal.execution` and
`signal.execution.failed` events there with `x-echozero-timestamp` (ISO, same as the
payload `timestamp`) and `x-echozero-signature` =
`HMAC_SHA256(webhookSecret, timestamp + "." + rawJsonBody)`. This is **not** the
inbound canonical-JSON scheme. Single attempt, 10 second timeout, no retries, so
handle duplicates by `signalId`. Make clear: inbound (their signals → EchoZero)
needs **no** hosting, they POST; outbound (EchoZero → them) needs an endpoint they
host. Skip if they don't have one yet.

### Step 6: Go live and what you can do now

Point them to the portal for the go-live calls that deserve a human and a screen,
and **don't flip these for them**:
> "When you're ready to go live, three things in the portal. I'll leave these to you:
> 1. **Ask for a listing.** Public listing goes through a staff review.
> 2. **Set your fee model.** Success fee = fixed 8% EchoZero + your 0 to 22%,
>    capped at 30% of new profit above each subscriber's high-water mark, settled
>    weekly. No per-trade fee on agent trades. Subscriptions: you keep 80% after
>    card fees.
> 3. **Configure exits:** TP ladder, stop-loss, leverage, mode.
> 👉 {ECHOZERO_PORTAL_URL}/agents/{id}
> (prod: `https://devportal.echozero.app/agents/{id}`)
> Reference docs: `https://docs.echozero.app` · API reference: `https://docs.echozero.app/api-reference`
> (raw OpenAPI spec: `https://mcp.echozero.app/api/docs-json`)"

Before the listing request, remind them: test in the sandbox and with virtual
wallet mode first, and keep the agent private until they have.

Then surface the rest of the suite so it gets discovered:
> "And from here, ask me anytime. I'm connected to your EchoZero account:
> • *'how's my agent doing?'* → portfolio & P&L
> • *'what's trending on Solana?'* → market data
> • *'send a buy signal for …'* → fire a signal
> • *'claim my fees'* → payouts
> (Powered by the **agent-manager**, **portfolio-check**, **signal-send**, and
> **market-data** skills, plus the `portfolio_overview` / `token_research` /
> `analyze_strategy_performance` prompts.)"

End on the win and the open door, never a dead stop.

---

## Failure & edge handling

- **401 mid-flow** → token expired (OAuth tokens last ~1h). Re-run
  `npx @echozero/ez login` (or re-check `ECHOZERO_API_KEY` **and** that calls are
  signed with `ECHOZERO_API_SECRET`; clock drift over 5 minutes also fails), resume
  the exact step. Don't restart.
- **404 on a call** → check the path against the docs index
  (`https://docs.echozero.app/llms.txt`) and the API spec
  (`https://mcp.echozero.app/api/docs-json`), correct it, retry once. Still
  failing → name the failed call and stop; don't loop.
- **Signal 404 "Developer agent not found"** → check the agent id.
- **Create agent 400** → name over 20 characters, or `HYPERLIQUID` without
  `signalExecutionDefaults.marketType: "perpetual"`. Fix and retry once.
- **Lost webhook secret** → `PATCH {BASE}/agents/:id`
  `{ rotateInboundWebhookSigningSecret: true }`, surface the new one once, note the
  old one is now dead.
- **Signal-group operator, not a developer** → switch to the
  **echozero-signal-group-setup** skill. You create the agent, fetch the connect
  link, verify the channel and read the parse log. The one step you can't do is
  adding the EchoZero bot: Telegram and Discord only let a channel or server admin
  do that in their own app or browser, so hand them the one-tap link and verify
  straight after. Don't claim you added it.
- **CI / non-interactive** → never block on a browser. Use `ECHOZERO_API_KEY` and
  `ECHOZERO_API_SECRET` throughout (signed), skip the `npx @echozero/ez login` and
  MCP-connect prose.

---

## Appendix A: Starter strategy scaffold (fallback for 5a)

> Use only if `generate_strategy` isn't reachable. Fill `ECHOZERO_AGENT_ID`,
> `ECHOZERO_INGRESS_URL`, `ECHOZERO_WEBHOOK_SECRET` from Step 3. Sends a
> **live**, HMAC-signed signal using the inbound signer in *Appendix E2*
> (save it as `ez-inbound.mjs` or `ez_inbound.py` next to this file).
>
> **This executes for subscribers. Test in the sandbox and with virtual wallet mode
> first, and keep the agent private until you have.** The scaffold asks for a typed
> confirmation before each send; keep that check unless the developer removes it
> deliberately.

### Node (default)

```js
import readline from "node:readline/promises";
import { sendSignal } from "./ez-inbound.mjs";

// This executes for subscribers. Test in the sandbox and with virtual wallet
// mode first, and keep the agent private until you have.
async function confirmLive(summary) {
  const rl = readline.createInterface({ input: process.stdin, output: process.stdout });
  const answer = await rl.question(`Send LIVE: ${summary}. Type "send" to confirm: `);
  rl.close();
  return answer.trim() === "send";
}

// --- your edge here -------------------------------------------------
// Solana spot buy. For Hyperliquid use chain "hyperliquid", instrument { coin: "ETH" },
// tradeType "perp", side "long" | "short", and leverageX.
const signal = {
  eventType: "buy",
  idempotencyKey: `my-strategy-${Date.now()}`, // stable per trade idea, reused on retry
  chain: "solana",
  instrument: { tokenAddress: "So11111111111111111111111111111111111111112" },
  symbol: "SOL",
  side: "long",
  tradeType: "spot",
  amount: 25,
  reasoning: "example signal",
  confidence: 0.6,
  triggers: { slPrice: { value: 130, targetType: "price" } },
};

if (await confirmLive(`buy ${signal.symbol} $${signal.amount}`)) {
  console.log(await sendSignal(signal));
} else {
  console.log("Not sent.");
}
```

### Python (on request)

```python
from ez_inbound import send_signal

# This executes for subscribers. Test in the sandbox and with virtual wallet
# mode first, and keep the agent private until you have.
def confirm_live(summary):
    return input(f'Send LIVE: {summary}. Type "send" to confirm: ').strip() == "send"

# --- your edge here ---
signal = {
    "eventType": "buy",
    "idempotencyKey": "my-strategy-0001",  # stable per trade idea, reused on retry
    "chain": "solana",
    "instrument": {"tokenAddress": "So11111111111111111111111111111111111111112"},
    "symbol": "SOL", "side": "long", "tradeType": "spot",
    "amount": 25, "reasoning": "example signal", "confidence": 0.6,
    "triggers": {"slPrice": {"value": 130, "targetType": "price"}},
}

if confirm_live(f"buy {signal['symbol']} ${signal['amount']}"):
    print(send_signal(signal))
else:
    print("Not sent.")
```

---

## Appendix B: Inbound webhook contract (authoritative)

Source: `https://docs.echozero.app/guides/webhook-security` and
`https://docs.echozero.app/guides/signal-envelope`.

- **Endpoint:** `POST {ECHOZERO_INGRESS_URL}` (= `https://mcp.echozero.app/api/public/agent-signals/:agentId`).
  Per-agent **HMAC** auth with the `ezw_…` secret; no API key on this call.
- **Headers:** `Content-Type: application/json`; `X-EZ-Timestamp` = unix
  **seconds** (±5 minutes skew, else 401); `X-EZ-Signature` = **lowercase hex**
  `HMAC_SHA256(secret, "{ts}.{canonicalJson}")`.
- **Canonical JSON:** keys sorted alphabetically at every level, `undefined`
  omitted, compact, numbers formatted the JavaScript way. The server drops unknown
  top-level fields before it verifies, so only `text`, a non-empty
  `idempotencyKey`, and the structured keys listed in Appendix E2 are signed. Send
  only documented fields.
- **Body, one style per request** (`text`, `eventType`, or legacy `action`):
  - **Structured envelope (preferred):** `eventType` is one of `trade_idea`, `buy`,
    `scale_in`, `sell`, `partial_sell`, `amend`, `breakeven`, `cancel`,
    `position_update`, `trade_review`. Required with `eventType`: `idempotencyKey`
    (≤128) and `reasoning` (shown in the public feed). Entries (`buy`,
    `scale_in`) carry `chain` (`solana` | `hyperliquid`), `instrument`
    (`{ tokenAddress }` for Solana, `{ coin }` or `{ market }` for Hyperliquid),
    `side` (`long` | `short`), `tradeType` (`spot` | `perp` | `virtual`), `amount`
    (USD), `leverageX` (perps) and `triggers` (`slPrice`, `tps[]`, `timeStopMins`).
    Lifecycle events (`sell`, `partial_sell`, `amend`, `breakeven`, `cancel`) need
    `positionRef` (a trade id or the entry's original `idempotencyKey`); `partial_sell`
    also takes `sellSizePct`, `amend` takes `changes`. Full field list: the *Signal
    envelope* guide.
  - **Natural language:** `text`, parsed by EchoZero. Always send an
    `idempotencyKey` too, so a retried request can't execute twice.
  - **Legacy:** `action` (`buy`|`sell`) + `tokenAddress`, used only when `eventType`
    is omitted. Don't teach this to new users.
- **Response:** `data.outcome` ∈ `matched | unmatched | skipped | failed | error`, plus
  `signalId`, `status`, `skipReason`. **401** bad signature or skew · **404**
  Developer agent not found (check the agent id) · **400** validation.
- Structured payloads **skip** NL parsing and go straight to execution; `text`
  payloads run the agent's parser first.

> 📝 **Note:** the inbound URL is `/api/public/agent-signals/<agentId>` with an `ezw_…`
> secret, tied to an *agent*. If you see a `whsec_…` secret or a
> `/v1/signals/<strategy-id>` URL, that is not the inbound endpoint, so don't
> sign against it. Never put REST API headers (`x-signature` / `x-timestamp`) on
> this call.

---

## Appendix C: Endpoint / tool cheat sheet (verify against the API spec)

| Purpose | REST call | MCP tool (if connected) |
|---|---|---|
| Verify auth / identity | `GET {ECHOZERO_MCP_URL}/status` · `GET {BASE}/users/me` | `get_current_user` |
| Register as developer | `POST {ECHOZERO_MCP_URL}/api/developers/register` (409 if already) | none |
| List my agents (dedupe) | `GET {BASE}/agents` | `get_my_agents` |
| Create agent | `POST {BASE}/agents` | `create_agent` |
| Update / set outbound webhook / rotate secret | `PATCH {BASE}/agents/:id` | `update_agent` |
| Persona validate / preview / save | `POST {BASE}/agents/:id/persona/validate` · `…/persona/preview` · `…/persona` | `validate_agent_persona`, `preview_agent_persona`, `save_agent_persona` |
| Generate a strategy | `POST {BASE}/ai/strategy/generate` (scope `write:ai`) | `generate_strategy` |
| Trending token for the test | `GET {BASE}/tokens/trending` | `get_trending_tokens` |
| Sandbox signal test (dry run) | `POST {ECHOZERO_MCP_URL}/api/sandbox/agents/:id/signals/simulate` | none |
| Sandbox trade test (dry run) | `POST {ECHOZERO_MCP_URL}/api/sandbox/agents/:id/trade` | none |
| Sandbox history | `GET {ECHOZERO_MCP_URL}/api/sandbox/agents/:id/signals` | none |
| Virtual wallet mode + funds | `POST {BASE}/wallet/mode` · `POST {BASE}/wallet/virtual/add-funds` | `toggle_wallet_mode`, `add_virtual_funds` |
| Live signal (HMAC, executes) | `POST {ECHOZERO_INGRESS_URL}` | none |
| Portfolio / P&L | `GET {BASE}/wallet/*`, `GET {BASE}/trades` | `get_wallet_mode`, `get_trades` |
| Claim fees | `POST {BASE}/developers/claims/agents/:id` · `…/claims/all` | `claim_fees` |

Auth: `Authorization: Bearer <oauth-token>` (interactive, no signing) **or**
`x-api-key: $ECHOZERO_API_KEY` **plus** `x-timestamp` and `x-signature` from
Appendix E1 (headless). The inbound signal call uses the per-agent HMAC from
Appendix E2, not either.

---

## Appendix D: Signal-relay adapter (for users who already have signals)

> For the **signal-relay** path. They don't need a strategy; they need to map
> their existing signal onto EchoZero's envelope and sign it. Adapt field names to
> their source.
>
> **This executes for subscribers. Test in the sandbox and with virtual wallet mode
> first, and keep the agent private until you have.** Wire it to the live ingress
> only after the developer has confirmed they are ready, and keep a confirmation
> step (or a dry-run flag that defaults to on) until they remove it deliberately.

```js
import { sendSignal } from "./ez-inbound.mjs"; // Appendix E2

// Stays in dry-run until the developer sets ECHOZERO_LIVE=yes on purpose.
// This executes for subscribers. Test in the sandbox and with virtual wallet
// mode first, and keep the agent private until you have.
const LIVE = process.env.ECHOZERO_LIVE === "yes";

// Map YOUR existing signal object onto EchoZero's envelope (Solana spot buy).
function toEchoZero(mySignal) {
  return {
    eventType: "buy",
    idempotencyKey: String(mySignal.id),       // required; dedupes retries
    chain: "solana",
    instrument: { tokenAddress: mySignal.mint }, // ← your field
    side: "long",
    tradeType: "spot",
    amount: mySignal.usdSize,                    // USD
    confidence: mySignal.score,                  // optional, 0-1
    reasoning: mySignal.note,                    // required, shown in the feed
  };
}

// In your existing "on new signal" handler:
export async function relay(incoming) {
  const signal = toEchoZero(incoming);
  if (!LIVE) return console.log("[dry-run] would send", signal);
  return sendSignal(signal);
}
```

`idempotencyKey` and `reasoning` are required for `eventType` signals. Use their
own signal id as the key so retries don't double-fire, and remind them `reasoning`
shows publicly in the agent's feed. For exits, send `eventType: "sell"` with
`positionRef` set to the entry's `idempotencyKey`.

---

## Appendix E: Signing helpers (copy these, don't re-derive them)

### E1: REST calls with an API key (headless)

Every REST call authenticated with `ECHOZERO_API_KEY` must carry `x-timestamp`
(epoch **milliseconds**) and `x-signature` = lowercase hex
`HMAC_SHA256(ECHOZERO_API_SECRET, timestamp + METHOD + path + body)`. `path` is the
full request path including any query string (for example `/api/v1/agents`), `body`
is the compact JSON you send (the server signs JSON.stringify of the parsed body; empty for GET). Drift over 5 minutes is rejected.
OAuth sessions don't sign.

Node (`ez-rest.mjs`):

```js
import { createHmac } from "node:crypto";

const BASE = process.env.ECHOZERO_MCP_URL || "https://mcp.echozero.app";

// path = full request path including any query string, e.g. "/api/v1/agents"
export function signRest(method, path, body = "", ts = String(Date.now())) {
  const sig = createHmac("sha256", process.env.ECHOZERO_API_SECRET)
    .update(ts + method.toUpperCase() + path + body)
    .digest("hex");
  return { "x-api-key": process.env.ECHOZERO_API_KEY, "x-timestamp": ts, "x-signature": sig };
}

// extraHeaders are sent but not signed, e.g. { "x-ez-mcp-client": "echozero-skills" }.
// The body is compact JSON.stringify output, and the same string is signed and sent.
export async function ez(method, path, json, extraHeaders = {}) {
  const body = json === undefined ? "" : JSON.stringify(json);
  const res = await fetch(BASE + path, {
    method,
    headers: { "Content-Type": "application/json", ...extraHeaders, ...signRest(method, path, body) },
    body: body || undefined,
  });
  return { status: res.status, data: await res.json().catch(() => null) };
}
```

Use it as `await ez("POST", "/api/v1/agents", { name: "…", agentVenue: "JUPITER", signalSourceKind: "webhook", isPublic: false }, { "x-ez-mcp-client": "echozero-skills" })`
or `await ez("GET", "/api/v1/agents")`. The Python `ez()` below takes the same optional
`extra_headers` argument.

Python needs two small helpers first, because `json.dumps` formats numbers differently
from JavaScript (`25.0`, `2e-05`) and the server signs the JavaScript form. `js_number`
matches `Number.prototype.toString` and `js_dumps` is compact `JSON.stringify` with real
UTF-8. Save this as `ez_json.py`; the inbound signer in E2 uses it too.

Python (`ez_json.py`):

```python
import json, math
from decimal import Decimal

def js_number(x):
    """Format a number the way JavaScript's Number.prototype.toString does."""
    if isinstance(x, int):
        if abs(x) < 2**53:
            return str(x)
        x = float(x)  # JS has no big ints: 10**21 prints as 1e+21
    if not math.isfinite(x):
        raise ValueError("numbers must be finite")
    if x == 0:
        return "0"
    r = repr(float(x))
    if "e" not in r:
        return r[:-2] if r.endswith(".0") else r
    if 1e-6 <= abs(x) < 1e21:  # JS never uses an exponent in this range
        return format(Decimal(r), "f")
    mant, exp = r.split("e")
    mant = mant[:-2] if mant.endswith(".0") else mant
    return f"{mant}e{'+' if int(exp) >= 0 else '-'}{abs(int(exp))}"

def js_dumps(v):
    """Compact JSON like JSON.stringify: no spaces, real UTF-8, JS number format."""
    if v is None:
        return "null"
    if isinstance(v, bool):
        return "true" if v else "false"
    if isinstance(v, (int, float)):
        return js_number(v)
    if isinstance(v, str):
        return json.dumps(v, ensure_ascii=False)
    if isinstance(v, (list, tuple)):
        return "[" + ",".join(js_dumps(x) for x in v) + "]"
    if isinstance(v, dict):
        return "{" + ",".join(json.dumps(k, ensure_ascii=False) + ":" + js_dumps(x) for k, x in v.items()) + "}"
    raise TypeError(f"cannot serialise {type(v).__name__}")
```

Python (`ez_rest.py`):

```python
import hashlib, hmac, json, os, time, urllib.request, urllib.error
from ez_json import js_dumps

BASE = os.environ.get("ECHOZERO_MCP_URL", "https://mcp.echozero.app")

def sign_rest(method, path, body="", ts=None):
    """path = full request path including any query string, e.g. /api/v1/agents"""
    ts = ts or str(int(time.time() * 1000))
    payload = (ts + method.upper() + path + body).encode()
    sig = hmac.new(os.environ["ECHOZERO_API_SECRET"].encode(), payload, hashlib.sha256).hexdigest()
    return {"x-api-key": os.environ["ECHOZERO_API_KEY"], "x-timestamp": ts, "x-signature": sig}

def ez(method, path, payload=None, extra_headers=None):
    # extra_headers are sent but not signed, e.g. {"x-ez-mcp-client": "echozero-skills"}
    body = "" if payload is None else js_dumps(payload)  # the same string is signed and sent
    req = urllib.request.Request(
        BASE + path, data=body.encode() or None, method=method,
        headers={"Content-Type": "application/json", **(extra_headers or {}), **sign_rest(method, path, body)})
    try:
        with urllib.request.urlopen(req) as r:
            return r.status, json.loads(r.read() or "null")
    except urllib.error.HTTPError as e:
        return e.code, json.loads(e.read() or "null")
```

Self-check (from the published test vectors): secret `test_secret`, timestamp
`1710000000000`, `POST`, path `/api/api-keys`, body `{"name":"SDK HMAC Test"}`
must give `274c9ff280eadf751530e9e7fce2c2a573d8676b13b7108fe353407df7cc9e00`.

### E2: Inbound signal (per-agent secret `ECHOZERO_WEBHOOK_SECRET`)

Signature = lowercase hex `HMAC_SHA256(ezw_secret, "{unixSeconds}.{canonicalJson}")`.
The canonical JSON is **not** the whole body: it contains only `text`, a non-empty
`idempotencyKey`, and the keys in `SIGNED_KEYS` below, with keys sorted
alphabetically at every level. Other top-level keys are dropped from the signed
form, and the server drops unknown top-level fields before it verifies, so send
only documented fields. Timestamps outside ±5 minutes are rejected with 401. This
matches the docs' webhook security guide
(`https://docs.echozero.app/guides/webhook-security`), which also publishes golden
test vectors. Prefer the SDK if they can install it: `signInboundWebhook` from
`@echozero/sdk` (or `sign_inbound_webhook` in the Python package `echozero`, 0.2.0
or later) does exactly this.

Node (`ez-inbound.mjs`):

```js
import { createHmac } from "node:crypto";

// Only these top-level keys are signed. Any other key in the body is ignored
// by the server when it checks the signature (it is still sent in the body).
const SIGNED_KEYS = [
  "action", "amount", "caption", "chain", "changes", "clientTimestamp", "confidence", "context",
  "currentPnlPct", "entryPrice", "entryZone", "eventType", "exitPrice", "exitReason",
  "expiresAt", "ideaId", "instrument", "leverageX", "metadata", "orderType", "pnlPct",
  "positionRef", "reasoning", "relatedTradeId", "riskMgmt", "sellSizePct", "side",
  "symbol", "tokenAddress", "tradeType", "triggers", "urgency", "version",
];

function clean(v) {
  if (v === undefined) return undefined;
  if (typeof v === "number") return Number.isFinite(v) ? v : undefined;
  if (Array.isArray(v)) return v.map(clean).filter((x) => x !== undefined);
  if (v && typeof v === "object") {
    const out = {};
    for (const k of Object.keys(v).sort()) {
      const c = clean(v[k]);
      if (c !== undefined) out[k] = c;
    }
    return out;
  }
  return v;
}

export function canonicalSignalJson(body) {
  const o = {};
  if (body.text !== undefined) o.text = body.text;
  const idk = typeof body.idempotencyKey === "string" ? body.idempotencyKey.trim() : "";
  if (idk) o.idempotencyKey = idk;
  for (const k of SIGNED_KEYS) {
    const c = clean(body[k]);
    if (c !== undefined) o[k] = c;
  }
  return JSON.stringify(Object.fromEntries(Object.keys(o).sort().map((k) => [k, o[k]])));
}

// Returns the headers for the inbound call. Timestamp is unix SECONDS.
export function signSignal(body, secret = process.env.ECHOZERO_WEBHOOK_SECRET, ts = String(Math.floor(Date.now() / 1000))) {
  const sig = createHmac("sha256", secret).update(`${ts}.${canonicalSignalJson(body)}`).digest("hex");
  return { "X-EZ-Timestamp": ts, "X-EZ-Signature": sig };
}

export async function sendSignal(body) {
  const res = await fetch(process.env.ECHOZERO_INGRESS_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json", ...signSignal(body) },
    body: JSON.stringify(body),
  });
  return { status: res.status, data: await res.json().catch(() => null) };
}
```

Python (`ez_inbound.py`):

```python
import hashlib, hmac, json, math, os, time, urllib.request, urllib.error
from ez_json import js_dumps  # see E1; JS-style numbers, compact, real UTF-8

# Only these top-level keys are signed. Any other key in the body is ignored
# by the server when it checks the signature (it is still sent in the body).
SIGNED_KEYS = [
    "action", "amount", "caption", "chain", "changes", "clientTimestamp", "confidence", "context",
    "currentPnlPct", "entryPrice", "entryZone", "eventType", "exitPrice", "exitReason",
    "expiresAt", "ideaId", "instrument", "leverageX", "metadata", "orderType", "pnlPct",
    "positionRef", "reasoning", "relatedTradeId", "riskMgmt", "sellSizePct", "side",
    "symbol", "tokenAddress", "tradeType", "triggers", "urgency", "version",
]

_DROP = object()  # the JS signer drops non-finite numbers and undefined

def _clean(v):
    if isinstance(v, bool) or v is None or isinstance(v, str):
        return v
    if isinstance(v, (int, float)):
        return v if math.isfinite(v) else _DROP
    if isinstance(v, list):
        return [c for c in (_clean(x) for x in v) if c is not _DROP]
    if isinstance(v, dict):
        out = {}
        for k in sorted(v):
            c = _clean(v[k])
            if c is not _DROP:
                out[k] = c
        return out
    return _DROP

def canonical_signal_json(body):
    o = {}
    if "text" in body:
        o["text"] = body["text"]
    idk = body.get("idempotencyKey")
    if isinstance(idk, str) and idk.strip():
        o["idempotencyKey"] = idk.strip()
    for k in SIGNED_KEYS:
        if k in body:
            c = _clean(body[k])
            if c is not _DROP:
                o[k] = c
    return js_dumps({k: o[k] for k in sorted(o)})

def sign_signal(body, secret=None, ts=None):
    """Returns the headers for the inbound call. Timestamp is unix SECONDS."""
    secret = secret or os.environ["ECHOZERO_WEBHOOK_SECRET"]
    ts = ts or str(int(time.time()))
    msg = f"{ts}.{canonical_signal_json(body)}".encode()
    return {"X-EZ-Timestamp": ts,
            "X-EZ-Signature": hmac.new(secret.encode(), msg, hashlib.sha256).hexdigest()}

def send_signal(body):
    req = urllib.request.Request(
        os.environ["ECHOZERO_INGRESS_URL"], data=js_dumps(body).encode(), method="POST",
        headers={"Content-Type": "application/json", **sign_signal(body)})
    try:
        with urllib.request.urlopen(req) as r:
            return r.status, json.loads(r.read())
    except urllib.error.HTTPError as e:
        return e.code, json.loads(e.read() or "null")
```

Self-check (published vectors): secret `secret`, timestamp `1710000000`, body
`{eventType:"buy", idempotencyKey:"test-1", reasoning:"test", tokenAddress:"So111…112", amount:500}`
(plus any extra field) must give
`be420f61d91e6b871481774c62f972f0aafa6f5f1a727ba1e4a32558784f77c3`.

---
## Runner variants

One skill, three runners. The only difference is the sign-in trigger (Step 1):
- **Claude Code**: `npx @echozero/ez login --client claude` (MCP auto-wired into Claude's config).
- **Codex**: `npx @echozero/ez login --client codex` (writes Codex MCP config).
- **OpenClaw**: `npx @echozero/ez login --client openclaw` (writes OpenClaw config).
Everything else (the flow, copy, contracts, and the headless `ECHOZERO_API_KEY` +
`ECHOZERO_API_SECRET` path) is identical across all three. Don't fork this file per
runner; if the installer needs per-runner artifacts, generate them from this source.
(See CONTRIBUTING.md.)
