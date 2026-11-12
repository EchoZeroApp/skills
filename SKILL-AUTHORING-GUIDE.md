---
name: echozero-skill-authoring-guide
description: >-
  How to write and review an EchoZero developer skill so it matches the quality
  bar set by the first five (agent-setup, agent-manager, portfolio-check,
  signal-send, market-data). Run this before writing a new skill, and again as a
  QA gate before shipping. If you're an AI assistant asked to build an EchoZero
  skill, follow this end to end — especially the "verify against the code" step.
---

# EchoZero Skill Authoring Guide

This is the playbook for building EchoZero developer skills (the `npx skills add
EchoZeroApp/skills` package). It exists so every new skill matches the existing
ones without re-learning the lessons. Two ways to use it:

1. **Authoring** — read Parts 1–3, draft from the template, then run Part 5.
2. **Reviewing** — run the Part 4 checklist against a finished skill; it must pass
   every line before it ships.

> The single most important rule, learned the hard way: **verify every endpoint,
> field name, and enum value against the public API spec before you write it
> down. Do not guess.** Our first drafts had real bugs — `type` vs `isPublic`,
> `closeOwnerTrades` vs `closeActiveTrades`, an invented "send signal" endpoint —
> that only checking the spec caught. A plausible guess that ships is
> worse than a TODO.

---

## Part 1 — Principles (the why)

Every EchoZero skill should embody these. They're distilled from how the best
MCP/CLI onboarding flows (Stripe, GitHub, Vercel, Cloudflare, Helius, Colosseum)
actually work.

- **Do it for them, don't instruct.** If a step is an API/tool call you can make,
  make it. "I've created your agent" beats "here's how to create an agent."
- **Detect before you ask.** Check auth state and existing resources first; never
  re-ask what you can look up. Returning users land mid-flow.
- **Fastest path to first success.** Get the user to one working result early
  (a created agent, a parsed signal, a real number), then offer depth. Defer
  config, code, and choices until after the first win.
- **Sane defaults offered as confirmations.** Propose the obvious answer ("I'll
  make it private while you test — ok?") so "yes" is the whole interaction.
- **Confirm before anything outward, irreversible, or money-moving — and show the
  impact first.** Reads are free. Claims, live signals, going public, deleting,
  closing trades: state the number / blast radius (e.g. subscriber count), then
  require an explicit yes. Never act on an implied yes.
- **Always leave a headless path.** Whatever browser/OAuth flow you offer, also
  support `ECHOZERO_API_KEY`. Never dead-end a VPS/CI user.
- **Every step ends on a visible win.** "✅ Signed in", "✅ Agent created",
  "✅ Test signal parsed." Users need to feel the loop close.
- **Summarize, don't dump.** Headline number first, then a tight breakdown. Never
  paste raw JSON at the user.
- **Be honest about scope.** Promise only what's built. If something the user
  expects isn't exposed (e.g. RSI/sentiment), say so — don't fabricate or imply.
- **Surface what they unlocked.** Sign-in connects MCP tools; setup reveals sibling
  skills. End by telling them what they can now do.
- **Use the platform's own capabilities.** Reach for built tools (`generate_strategy`,
  persona endpoints, MCP tools) instead of reinventing them in the skill.

---

## Part 2 — Required structure (the template)

Every skill is one Markdown file at `<skill-name>/SKILL.md` (the layout the
`npx skills` installer discovers), with these sections in this order. Consistency
across the suite is itself a quality requirement — match the existing files' shape
and tone.

1. **Frontmatter** — `name` (kebab, `echozero-…`) and a `description` that is
   trigger-rich: weave in the exact phrases a user would say ("claim my fees",
   "what's trending"). The description is how the skill gets invoked.
2. **Title + intro** — one paragraph on what the skill does and the model behind it
   (developers send signals off-platform; EchoZero is custodial and multi-venue —
   Solana spot (Jupiter) + Hyperliquid perps; two venues, each agent locked to one at create).
3. **Prime directive** — the one safety/UX sentence that governs this skill
   (e.g. "treat a live send like spending other people's money").
4. **Golden rules** — the Part 1 principles, tailored to this skill.
5. **Configuration** — the standard block (Appendix A). Copy it verbatim; never
   re-derive the URL layout.
6. **Step 0 — authenticate** — `GET {ECHOZERO_MCP_URL}/status`; on 401 route to
   echozero-agent-setup; then detect existing resources.
7. **Operations / flow** — the actual work, one step at a time, each with: what to
   do, the exact endpoint/tool, the confirm policy, and the visible win.
8. **Failure & edge handling** — 401 (re-auth, resume), 404 (check llms.txt), plus
   the skill-specific failure modes in plain language.
9. **"What else they can do"** — cross-sell the sibling skills so the suite is
   discoverable. Never dead-end.
10. **Endpoint / tool cheat sheet** — a table of REST call + MCP tool per operation.
11. **Runner variants** — see Part 6.

---

## Part 3: EchoZero API facts (canonical; don't guess these)

Verify against the public API spec (`https://mcp.echozero.app/api/docs-json`) and the
docs (`https://docs.echozero.app`). Re-verify if the API changes.

**URL layout** (the part everyone gets wrong):
- Server sets a global `/api` prefix, but **excludes `/status` and `/oauth/*`** —
  those live at the **server root**: `{ECHOZERO_MCP_URL}/status`,
  `{ECHOZERO_MCP_URL}/oauth/*`, `{ECHOZERO_MCP_URL}/mcp`.
- REST sits under `/api/v1` → `ECHOZERO_API_BASE = {ECHOZERO_MCP_URL}/api/v1`
  (prod `https://mcp.echozero.app/api/v1`; `api.echozero.app` is a **different,
  live** EchoZero host — it responds 200 at its root, but dev API paths 404 there,
  so never use it as a fallback).
- **Under `/api` but NOT `/v1`** (call at `{ECHOZERO_MCP_URL}/api/…`):
  `developers/me`, `api-keys`, `dev/trade-signals`, `developers/signals`,
  `public/agent-signals/:id`.

**Auth**: `npx @echozero/ez login` (OAuth 2.1 + PKCE, flag `--client claude|codex|openclaw`,
stores token at `~/.echozero/ez-login.json` and **also wires the MCP server** into
the client), or `ECHOZERO_API_KEY` as `x-api-key` for headless (REST calls with a key must also
be HMAC signed with `ECHOZERO_API_SECRET`: `x-timestamp` in epoch ms + `x-signature`
= `HMAC_SHA256(secret, timestamp + METHOD + path + body)`). Verify at
`/status`. Tokens last ~1h. `npx @echozero/ez login` defaults to `https://mcp.echozero.app`.

**Agent fields / enums**: visibility is **`isPublic`** (boolean), not `type`.
`AgentStatus` = `DRAFT | ACTIVE | PAUSED | DELETED`. Pause flag is
**`closeActiveTrades`** (required for private agents). Source kind is
`signalSourceKind: "webhook"`. Create/update via `POST` / `PATCH /v1/agents/:id`.

**Money / claims**: check first `GET /v1/developers/earnings`
(`totalClaimableUsd` + breakdown) and `GET /v1/developers/claims/strategy-fees`
(per-agent split — note there is no bare `GET /v1/developers/claims`, it 404s).
Claim trading fees per-agent `POST /v1/developers/claims/agents/:id` or all at once
`POST /v1/developers/claims/all`, **lands as USDC in the developer's EchoZero
wallet**, possibly capped per agent per call. Subscription earnings:
`POST /v1/developers/claim-subscription-earnings`. Needs scope `read:earn`. A payout
address is optional and Stripe Connect is retired, so never gate a claim on either.
Always show the claimable amount before claiming.

**Signals**: inbound is `POST /public/agent-signals/:agentId`, **per-agent HMAC**
(`X-EZ-Timestamp` + `X-EZ-Signature` = lowercase hex `HMAC_SHA256(secret,
"{unixSeconds}.{canonicalJson}")`; canonical = keys sorted at every level,
undefined dropped, finite numbers; secret `ezw_…` shown once on create) — **this
executes**. Sandbox is `POST /api/dev/trade-signals` (no webhook HMAC, no execution; the
documented sandbox routes are `/api/sandbox/agents/:id/signals/simulate` and
`…/trade`). Live body: the structured envelope (`eventType`, `idempotencyKey` and
`reasoning` required; `chain`, `instrument`, `side`, `tradeType`, `leverageX`,
`triggers`, `positionRef` for lifecycle events) **or** `text`, never both; the
`action`+`tokenAddress` shape is legacy. The signature covers only the fields the
server signs (unknown top-level keys are dropped from the canonical JSON); use the
helper in echozero-agent-setup Appendix E2. **There is no authenticated
POST-to-send endpoint** (`developers/signals` is GET-only).

**Market data**: `GET /v1/tokens/trending`, `GET /v1/tokens` (filter/search via
`?query=…`), `GET /v1/tokens/:idOrSymbol` (Mongo `_id`, case-insensitive symbol,
or mint/contract address; returns the mint), `GET /v1/ai/trades/quote` (`action` BUY|SELL, `tokenAddress`, `amountUsd` required). **No RSI /
technical-indicator or sentiment/Fear-&-Greed endpoints**; don't promise them.

**Always prefer connected MCP tools** (`get_my_agents`, `create_agent`,
`claim_fees`, `get_trending_tokens`, …) when present; REST is the fallback that
works in any runner and the only path in headless mode.

---

## Part 4 — QA checklist (the gate — every line must pass)

Run this as an adversarial self-critique on the finished skill.

**Accuracy**
- [ ] Every endpoint, field, and enum verified against the public API spec and docs, not guessed.
- [ ] No "confirm path" / "TODO" placeholders left (resolve them by checking the API spec).
- [ ] Correct `/v1` vs `/api` (vs server-root) for every path.

**Safety**
- [ ] Every money-moving / destructive / outward action requires an explicit yes.
- [ ] The impact is shown *before* the confirm (amount, subscriber count, what closes).
- [ ] Sandbox/dry-run is the default where one exists.

**Experience**
- [ ] Fastest reasonable path to first success; depth deferred.
- [ ] One step at a time; sane defaults offered as confirmations.
- [ ] Summarizes results; never dumps raw JSON. Visible win per step.

**Honesty**
- [ ] Promises only what's built; explicitly flags what isn't available.

**Headless**
- [ ] `ECHOZERO_API_KEY` path offered everywhere OAuth is; no dead-ends.

**Voice**
- [ ] Second person, imperative, concise. **No "easy", "simply", "just", "quick".**

**Built-tech**
- [ ] Uses platform capabilities instead of reinventing; announces what login unlocked.

**Consistency**
- [ ] Same structure, Configuration block, and cheat-sheet format as the other skills.
- [ ] Runner variants covered (Part 6).

**Cross-sell**
- [ ] Ends by surfacing the sibling skills.

---

## Part 5 — Process (how to actually build one)

1. **Read** this guide and one existing skill (`agent-setup/SKILL.md` is the
   reference) before writing.
2. **Draft** from the Part 2 template.
3. **Verify against the spec.** For every endpoint/field/enum, check the public API spec
   (`https://mcp.echozero.app/api/docs-json`) and `https://docs.echozero.app` and confirm the exact name and path. This
   step is non-negotiable and is where the bugs hide.
4. **Self-QA.** Run the Part 4 checklist as if you were trying to break the skill.
   Write down the issues — don't fix as you go.
5. **Fix** based on the list, then re-verify the lines you touched.
6. **Only then ship.** A skill that hasn't been spec-verified and checklist-QA'd is
   not done.

---

## Part 6 — Runner variants (DRY rule)

The three runners (Claude Code, Codex, OpenClaw) differ in **exactly one thing**:
the `npx @echozero/ez login --client <runner>` flag (the CLI writes the right per-runner config,
including the MCP wiring). Everything else — flow, copy, contracts — is identical.

**So: one file per capability, covering all three runners inline.** Do **not**
hand-maintain three near-duplicate files per skill — that recreates the drift this
guide exists to prevent (a contract fix would have to land in 15 files instead of
5). State the per-runner trigger in the skill's "Runner variants" section. If the
packaging/installer genuinely needs separate per-runner artifacts, **generate them
mechanically at build time** from the single source — never by hand.

---

## Appendix A — Standard Configuration block (copy verbatim)

```markdown
## Configuration

Read from the environment; do not hardcode:

- `ECHOZERO_MCP_URL` — host root of the MCP/OAuth server. Auth/verify live here at
  the root (`/status`, `/oauth/*`, `/mcp` — excluded from the `/api` prefix).
  Production is `https://mcp.echozero.app` (verified live).
- `ECHOZERO_API_BASE` — REST base = `{ECHOZERO_MCP_URL}/api/v1` (prod
  `https://mcp.echozero.app/api/v1`; `api.echozero.app` is a **different, live**
  EchoZero host — dev API paths 404 there, never use it as a fallback). Exceptions
  under `/api` but NOT `/v1` (`{ECHOZERO_MCP_URL}/api/…`): `developers/me`, `api-keys`,
  `dev/trade-signals`, `developers/signals`, `public/agent-signals/:id`.
- `ECHOZERO_API_KEY` — headless auth; send as `x-api-key`.
- `ECHOZERO_API_SECRET`, the key's secret. Headless REST calls must be HMAC signed
  (`x-timestamp` ms + `x-signature`); copy the helper from echozero-agent-setup
  Appendix E1.

> ✅ Host confirmed live: `mcp.echozero.app` is listed as the REST/MCP host in
> `https://echozero.app/llms.txt`; the `/v1` suffix is confirmed by direct endpoint
> testing, not that file's shorthand (it just says `.../api`). `api.echozero.app`
> is a different, live EchoZero host — dev API paths 404 there. Auth/MCP live at
> the root.

Prefer connected MCP tools when present; REST is the any-runner fallback.
```

## Appendix B — Runner variants section (copy, fill the runner deltas)

```markdown
## Runner variants

One skill, three runners. The only difference is the sign-in trigger:
- **Claude Code** — `npx @echozero/ez login --client claude`; MCP auto-wired into Claude's config.
- **Codex** — `npx @echozero/ez login --client codex`; writes Codex MCP config.
- **OpenClaw** — `npx @echozero/ez login --client openclaw`; writes OpenClaw config.
Flow, copy, and API contracts are identical. Do not fork this file per runner; if
the installer needs per-runner artifacts, generate them from this source.
```

## Appendix C — Canonical HMAC signing (for any skill that sends live signals)

```ts
import { createHmac } from "node:crypto";
function canonical(o: unknown): string {
  if (Array.isArray(o)) return `[${o.map(canonical).join(",")}]`;
  if (o && typeof o === "object") {
    const k = Object.keys(o as object).filter(x => (o as any)[x] !== undefined).sort();
    return `{${k.map(x => `${JSON.stringify(x)}:${canonical((o as any)[x])}`).join(",")}}`;
  }
  return JSON.stringify(o);
}
const ts = Math.floor(Date.now() / 1000).toString();
const sig = createHmac("sha256", SECRET).update(`${ts}.${canonical(body)}`).digest("hex");
// POST to the agent ingress URL with headers X-EZ-Timestamp: ts, X-EZ-Signature: sig
```
