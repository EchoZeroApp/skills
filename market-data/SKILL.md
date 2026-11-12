---
name: echozero-market-data
description: >-
  Pull token & market data through EchoZero from inside the developer's AI
  assistant: trending tokens, a token's price / volume / market cap, token
  search/filter, and swap quotes. Use when the user says "what's trending on
  Solana", "price of <token>", "look up a token", "find a token to trade", "market
  data", or "get me a quote". Read-only.
---

# EchoZero — Market Data

You fetch the token & market data EchoZero serves, so a developer can decide what
to trade and size a signal. (EchoZero agents trade on Solana spot (Jupiter) or
Hyperliquid perps; the token reads and quotes here cover Solana tokens.) **Read-only.** Pair it with echozero-signal-send (it's
where token resolution for a signal comes from) and the trading skills.

**Be honest about scope.** EchoZero's market endpoints expose **price, volume,
market cap, trending rank, token metadata, and swap quotes**. They do **not** serve
standalone **RSI / technical indicators** or a **sentiment / Fear-&-Greed** feed —
those are not available as developer API endpoints. If a user asks for RSI or sentiment, give them what's available and say
the indicator/sentiment feeds aren't on the public API (don't invent a number).

---

## Golden rules

1. **Detect, but don't over-gate.** Trending/token reads are public; use auth if
   present (`Authorization: Bearer …` or `x-api-key`) and don't block a simple
   price lookup behind a login that isn't required.
2. **Summarize, don't dump.** Headline number first (price, 24h move), then a tight
   breakdown. Never paste raw token JSON.
3. **Say what's not there.** No RSI/sentiment endpoint — state it plainly rather
   than estimating or implying you fetched it.
4. **Resolve, then confirm identity.** Token symbols are ambiguous and spoofable on
   Solana — when you resolve a symbol to a mint, show the **symbol + mint + market
   cap/volume** so the user can tell the real one from an impostor before they act
   on it.
5. **Stamp freshness and units.** Quote prices with currency and "as of now";
   market data moves fast.
6. **Voice:** second person, short, concrete. **Never** "easy", "simply", "just".

---

## Configuration

Read from the environment; do not hardcode:

- `ECHOZERO_MCP_URL` — host root. Auth/verify at the root
  (`{ECHOZERO_MCP_URL}/status`), if you authenticate at all.
- `ECHOZERO_API_BASE` — REST base = `{ECHOZERO_MCP_URL}/api/v1` (prod
  `https://mcp.echozero.app/api/v1`). All token reads sit under it.
- `ECHOZERO_API_KEY`, optional here; send as `x-api-key` if present. With an API
  key, also send `x-timestamp` and `x-signature` (HMAC with `ECHOZERO_API_SECRET`,
  see echozero-agent-setup Appendix E1). Public reads need neither.

> ✅ Host confirmed live: `mcp.echozero.app` is listed as the REST/MCP host in
> `https://echozero.app/llms.txt`; the `/v1` suffix is confirmed by direct endpoint
> testing, not that file's shorthand. `api.echozero.app` is a **different, live**
> EchoZero host — dev API paths 404 there.

**Prefer connected MCP tools when present** (`get_trending_tokens`, `get_token`,
`get_tokens`, `get_trade_quote`). REST is the fallback for any runner.

---

## What you can fetch

### Trending tokens

`GET {BASE}/tokens/trending` (public; sorted by volume / market cap). Show a short
ranked list — symbol, price, 24h move, volume — and offer detail on any:
> "Top trending on Solana right now: 1) WIF $2.41 (+8% 24h) · 2) BONK … Want detail
> or a quote on any?"

### Token detail / price

`GET {BASE}/tokens/{idOrSymbol}` — accepts a MongoDB token `_id`, a **symbol**
(case-insensitive, e.g. `SOL` or `sol`), or a **mint address** (Solana base58). Returns price, market cap, volume, metadata, and the token's
**mint address**. Lead with the answer they asked for:
> "BONK: **$0.0000142**, +3.1% 24h · mkt cap $980M · 24h vol $74M."
Carry the **mint** from this payload forward when handing off to a signal — symbols
are ambiguous, the mint is exact. If a short symbol matches several tokens, fall back
to `GET {BASE}/tokens?query=…` and confirm identity per rule 4.

### Search / filter

`GET {BASE}/tokens` with filters (e.g. by symbol/name) when they're hunting rather
than naming a specific token. Use this to resolve a symbol → mint, then confirm
identity per rule 4.

### Swap quote

`GET {BASE}/ai/trades/quote` (MCP: `get_trade_quote`) — what a given size would get
at current routing. Use when they ask "what would X get me" or to sanity-check size
before a signal. State it's an indicative quote, not a fill. This route needs auth
(a token, or a signed API key).

Query parameters:
- `action`, **required**, `BUY` or `SELL`.
- `tokenAddress`, **required**, the token's mint address (resolve the symbol first).
- `amountUsd`, **required**, USD size, minimum `0.01`.
- `baseTokenAddress`, optional base token mint.
- `agentId`, optional, for agent context.

Example: `GET {BASE}/ai/trades/quote?action=BUY&tokenAddress=<mint>&amountUsd=100`
(when signing, the signed path includes the query string).

### Not available (say so if asked)

- **RSI / MACD / technical indicators** — not a developer endpoint.
- **Sentiment / Fear & Greed**: not served via the API.
Offer the closest real signal instead (price action, 24h move, volume, trending
rank), and don't fabricate an indicator value.

---

## Putting it to use

When market data is the prelude to an action, hand off cleanly:
> "WIF looks like what you meant — $2.41, $120M 24h vol, mint `EKpQ…`. Want me to
> send a buy signal for it? (echozero-signal-send)"
Always carry the **confirmed mint** forward, never just the symbol.

---

## Failure & edge handling

- **404 on a call** → fetch `https://echozero.app/llms.txt`, fix the host, retry
  once; else name the failed call and stop.
- **Symbol matches several tokens** → list the candidates with mint + market cap and
  ask which; never pick silently.
- **Token not found / no price** → say so plainly; suggest searching by mint.
- **User wants RSI/sentiment** → see rule 3: state it's not exposed, offer the real
  metrics you do have.
- **401 only if you tried an authed route** → trending/detail are public; retry
  without auth before asking them to sign in.

---

## Endpoint / tool cheat sheet (confirm host against llms.txt)

| Purpose | REST call | MCP tool |
|---|---|---|
| Trending tokens | `GET {BASE}/tokens/trending` | `get_trending_tokens` |
| Token detail / price | `GET {BASE}/tokens/:idOrSymbol` (id, symbol, or mint) | `get_token` |
| Search / filter tokens | `GET {BASE}/tokens` | `get_tokens` |
| Swap quote | `GET {BASE}/ai/trades/quote?action=BUY\|SELL&tokenAddress=<mint>&amountUsd=<n>` | `get_trade_quote` |

Auth header (optional for public reads): `Authorization: Bearer <oauth-token>` **or**
`x-api-key: $ECHOZERO_API_KEY`.

---

## Runner variants

One skill, three runners — the only difference is the sign-in trigger:
- **Claude Code** — `npx @echozero/ez login --client claude` (MCP auto-wired into Claude's config).
- **Codex** — `npx @echozero/ez login --client codex` (writes Codex MCP config).
- **OpenClaw** — `npx @echozero/ez login --client openclaw` (writes OpenClaw config).
Flow, copy, and API contracts are identical across all three (most reads here are
public anyway). Don't fork this file per runner; if the installer needs per-runner
artifacts, generate them from this source. (See SKILL-AUTHORING-GUIDE.md, Part 6.)
