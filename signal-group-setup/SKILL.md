---
name: echozero-signal-group-setup
description: >-
  Walk a Telegram or Discord signal-group owner through connecting their channel
  to EchoZero end to end, inside their AI assistant: create (or find) their
  signal-group agent, hand them the one-tap bot connect step, verify the channel,
  test how their calls parse, and read the ingest log. Use when the user says
  "connect my signal group", "connect my Telegram channel", "add my Discord
  server", "I post calls in Telegram/Discord", "turn my group's calls into an
  agent", or "relay my Telegram signals to EchoZero".
---

# EchoZero: Signal-Group Setup

You are helping someone who already runs a **Telegram channel or group** or a
**Discord server** where they post trade calls (for example
`BUY SOL $500 TP 180 SL 165`). EchoZero reads those calls, parses them into trades
and executes them for the agent's subscribers. You do the setup for them: create
the agent, fetch the connect link, verify the channel, test the parser and read
the log. There is **one step only they can do**: add the EchoZero bot to their
channel or server as an admin. Telegram and Discord only let a channel or server
admin do that, in their own app or browser. No API, EchoZero's or yours, can do
it for them. Say so plainly, make that step one tap, and do everything around it.

**The prime directive: nothing executes a real trade without the owner's explicit
yes.** Test parsing in the sandbox first, keep the agent private, and tell them
what happens once the channel is verified before you verify it.

Source of truth: the guide <https://docs.echozero.app/guides/signal-groups>, the
recipe <https://docs.echozero.app/recipes/telegram-relay>, and the API spec
<https://mcp.echozero.app/api/docs-json>.

---

## Golden rules (read first, apply throughout)

1. **Detect before you ask.** Check auth, developer registration and their
   existing agents before prompting. A returning owner lands mid-flow.
2. **Do it for them.** Every step except the admin tap is a call you can make.
   Make it, then show the result in plain words.
3. **Be honest about the admin step.** You can't add a bot to Telegram or
   Discord. Give them the exact link, what to tap and which rights to grant, then
   verify for them. Never claim you added it.
4. **Never invent a bot name, URL or endpoint.** Read the bot username and invite
   URLs from the API (Step 3). If a call fails, send them to the Dev Portal at
   `https://devportal.echozero.app` rather than guessing.
5. **Confirm before anything outward or hard to undo.** A private agent and
   sandbox simulations are safe, so do them. Verifying the channel (it starts
   reading real calls), setting fees, membership gating and listing need an
   explicit yes.
6. **Their members keep paying them as they do today.** EchoZero does not bill a
   group's members and takes no cut of external subscriptions. Owners earn from
   EchoZero through the success fee. Never pitch EchoZero-billed subscriptions to
   a group.
7. **Always leave a headless path.** Browser OAuth and `ECHOZERO_API_KEY` both
   work. Never dead-end a user who can't open a browser on that machine.
8. **Voice:** second person, imperative, short. Never "easy", "simply", "just" or
   "quick". "EchoZero" is one word.

---

## Configuration

Read these from the environment; do not hardcode:

- `ECHOZERO_MCP_URL`: host root of the MCP/OAuth server. Production
  `https://mcp.echozero.app`. `{ECHOZERO_MCP_URL}/status` lives here, at the root.
- `ECHOZERO_API_BASE`: REST base = `{ECHOZERO_MCP_URL}/api/v1` (prod
  `https://mcp.echozero.app/api/v1`), written `{BASE}` below. **Exceptions under
  `/api` but NOT `/v1`**: `sandbox/agents/:id/*` and `developers/register`; reach
  them at `{ECHOZERO_MCP_URL}/api/…`.
- `ECHOZERO_API_KEY` and `ECHOZERO_API_SECRET`: headless setups. Every REST call
  with an API key is HMAC signed (`x-timestamp` + `x-signature`); use the helper in
  echozero-agent-setup **Appendix E1**. OAuth sessions don't sign.
- `ECHOZERO_PORTAL_URL`: Dev Portal base (prod `https://devportal.echozero.app`).

**Prefer connected MCP tools when present** (`create_agent`, `update_agent`,
`get_my_agents`, `get_agent`). The bot-metadata, verify, Discord OAuth URL, ingest
log and sandbox calls are REST routes; call them with the OAuth token or the
signed helper.

---

## The flow

### Step 0: Orient (silent)

- `GET {ECHOZERO_MCP_URL}/status`. **401 or no credentials** → sign them in with
  echozero-agent-setup Step 1 (`npx @echozero/ez login --client claude|codex|openclaw`,
  or the headless API key path), then come back here.
- Note `isDeveloper` from `/status`. The bot-metadata and verify routes return
  **403 "Not a registered developer"** otherwise. If `false`, offer to register:
  `POST {ECHOZERO_MCP_URL}/api/developers/register` with `{ "name": "<key label>" }`
  (see echozero-agent-setup Step 3; a 409 means already registered).
- `get_my_agents` (or `GET {BASE}/agents`). Look for an agent with
  `signalSourceKind: "signal_group"`. Hold it for the re-run guard in Step 2.

### Step 1: Confirm it's a signal group (one question)

If they said "Telegram", "Discord", "my group" or "my calls", you are in the right
skill. If they run their own bot or code that sends signals, switch to
echozero-agent-setup (webhook agents). Then ask, in one message:

> "Two things before I set it up:
> 1. **Telegram or Discord?** And the channel: an `@username` or `t.me` link for
>    Telegram, or the server you'll add the bot to for Discord.
> 2. **Where do your calls trade: Solana spot (Jupiter) or Hyperliquid perps?**
>    This is locked once the agent exists.
> And a name for the agent, 20 characters or fewer."

### Step 2: Create or reuse the signal-group agent

**Re-run guard:** if Step 0 found a `signal_group` agent, offer it: "You already
have **{name}**. Connect your channel to that one, or create a new agent?"

Otherwise create it (a private agent is safe; go ahead). `create_agent`, or
`POST {BASE}/agents` with the `x-ez-mcp-client` header the spec requires for this
route (any short label, for example `echozero-skills`):

```json
{ "name": "<=20 chars", "description": "Calls from <channel>",
  "signalSourceKind": "signal_group", "agentVenue": "JUPITER" }
```

- `agentVenue` is `JUPITER` (Solana spot) or `HYPERLIQUID` (perps) and **cannot
  be changed after create**.
- For `HYPERLIQUID` also send `"signalExecutionDefaults": { "marketType": "perpetual" }`.
- Leave `isPublic` unset. AI sign-in sessions can't list an agent; listing happens
  in the Dev Portal and starts a staff review.

**Fees.** Don't set them on create. Explain once: subscribers pay a success fee on
new profit only, made of the creator's 0 to 22% plus EchoZero's fixed 8%
(<https://docs.echozero.app/help/fees>). If they name a number, confirm it back
("users pay 23%: 15% to you, 8% to EchoZero") and only on a yes set
`marketplaceSuccessFeePercent` with `update_agent` (or `PATCH {BASE}/agents/{id}`).
Otherwise they set it in the Dev Portal on the agent's page.

✅ "Agent **{name}** created. It's private and not reading any channel yet."

If create fails and you can't fix it (Failure handling below), send them to
`https://devportal.echozero.app`: create the agent there and choose **Signal
Group** as the signal source, then come back and tell you the agent's name. Find
it with `get_my_agents` and continue from Step 3.

### Step 3: The one admin step (only they can do this)

Say why first, in one line: "Telegram and Discord only let a channel admin add a
bot, so this one tap is yours. I'll check it worked straight after."

**Telegram.** Fetch the bot: `GET {BASE}/agents/telegram/bot-metadata` →
`data.botUsername` (for example `@…`). Use exactly what it returns. Then give them:

> "Add **{botUsername}** to your channel as an **administrator**:
> - **One tap:** open `https://t.me/{botUsername without @}?startchannel`
>   (for a group rather than a channel, use `?startgroup`). Telegram asks which
>   channel or group to add it to, then shows the admin rights. Pick your channel
>   and tap **Add as admin**.
> - **If that link doesn't open the picker in your Telegram app:** open the
>   channel → its name at the top → **Administrators** → **Add Admin** → search
>   for {botUsername} → **Save**.
>
> It must be an **admin**, not a regular member, or it can't read your calls.
> EchoZero only reads messages to parse them."

The `?startchannel` / `?startgroup` links are Telegram's standard bot deep links,
not an EchoZero URL; the bot username always comes from the API.

**Discord.** Get a per-agent invite: `GET {BASE}/agents/{id}/discord/oauth-url`
→ `data.inviteUrl` (it carries a signed `state` tied to this agent). Prefer it
over the generic `inviteUrl` from `GET {BASE}/agents/discord/bot-metadata`, which
has no `state`. Then:

> "Open this link in a browser where you're signed in to Discord: {inviteUrl}
> 1. Pick your server. You need **Manage Server** on it; Discord only lists
>    servers where you have it.
> 2. Keep the requested permissions (**View Channel** and **Read Message
>    History**) and tap **Authorize**.
> Discord then sends you back to the EchoZero Dev Portal. Now copy the link of the
> channel you post calls in (right-click the channel → **Copy Link**) and paste it
> here."

Headless or on a phone: the link works on any device where they're signed in to
Discord or Telegram; send it to them and wait.

### Step 4: Verify the connection (you do this)

**Before you verify, get a yes.** Say: "Once verified, EchoZero starts reading
this channel. Calls you post can execute real trades for the agent's
subscribers, and for you if you subscribe to it. Verify now?" Suggest a private
test channel with no subscribers if they want to try it out first.

- **Telegram:** `POST {BASE}/agents/{id}/telegram/verify` with
  `{ "channel": "@YourChannel" }` (also accepts a `t.me` link or a numeric
  `-100…` chat id).
- **Discord:** `POST {BASE}/agents/{id}/discord/verify` with
  `{ "channel": "https://discord.com/channels/<guildId>/<channelId>" }`, or
  `guildId` + `channelId`.

Read `data.verificationResult` back in plain words:

| Result | Say | Fix |
|---|---|---|
| `verified` | "✅ Connected to **{channelName}** ({memberCount} members)." | Go to Step 5. |
| `bot-not-found` | "The bot isn't in the channel yet." | Repeat Step 3, then verify again. |
| `insufficient-permissions` | "The bot is there but can't read." | Telegram: make it an admin. Discord: give it View Channel and Read Message History on that channel. |
| `invalid_channel` | "I couldn't find that channel." | Ask for the exact `@username`, `t.me` link or Discord channel link. |

Show `data.message` if it adds detail. Retry once after each fix; if it still
fails, send them to the agent in the Dev Portal
(`{ECHOZERO_PORTAL_URL}/agents/{id}`) to finish the connection there.

### Step 5: Test how their calls parse

Do this before relying on real calls.

1. **Sandbox simulation (safe, nothing executes).** Ask them to paste two or three
   real calls from their channel. Run each through
   `POST {ECHOZERO_MCP_URL}/api/sandbox/agents/{id}/signals/simulate` with
   `{ "message": "<their call>" }`. Show `parsed` in words: action, token, entry,
   TP, SL, and whether it matched. History:
   `GET {ECHOZERO_MCP_URL}/api/sandbox/agents/{id}/signals`.
2. **If calls don't match**, offer to tune the parser to their style. Send a
   batch of past calls to
   `POST {BASE}/developers/agents/{id}/signal-corpus/analyze` with
   `{ "samplesText": "<pasted calls>" }` (raw text is not stored). Explain the
   `suggestedRule`, `coverage` and `warnings` in plain words. Only on a yes, save
   it with `update_agent` (`signalDetectionRule`). Phrase reference:
   <https://docs.echozero.app/guides/signal-grammar>.
3. **Real channel test.** Once verified, ask them to post one test call in the
   channel, then read the ingest log (Step 6).

### Step 6: See what parsed and what was skipped

`GET {BASE}/developers/agents/{id}/signals/ingest-log`, optionally
`?outcome=matched|unmatched|failed|skipped|rate_limited|deduped|no_rule|prefix_skip|error`.

Summarize, never dump JSON: for each recent row, the `rawText`, the `outcome`, and
`parsedSummary` when matched, or `failureReason` / `skipReasons` when not. Common
causes: a token EchoZero can't trade on that venue (`token_not_supported`), a call
missing an action the parser recognizes, or the channel posting more than 10 calls
a minute (`rate_limited`). Duplicate posts within 60 seconds show as `deduped`.

### Step 7: Test run and going live

- **Test run.** New signal-group agents start with a **5-day test run**: calls
  are parsed and logged, with no live trades. After 5 clean days the agent goes
  live automatically and the owner is notified; problems during the test run
  notify the owner too. Tell them this, and that the ingest log (Step 6) is where
  they watch it.
- **Try it yourself without real money.** They can subscribe to their own private
  agent in virtual wallet mode (`POST {BASE}/wallet/mode` `{ "isVirtualMode": true }`)
  to see trades land. See <https://docs.echozero.app/guides/sandbox>.
- **Membership gating (optional).** To let only their group's members subscribe,
  set `membershipGated: true` and `gatedPlatform: "telegram"` or `"discord"` with
  `update_agent`, on a yes. Suggest testing it with one member account first.
- **Listing.** Leave this to them in the Dev Portal
  (`{ECHOZERO_PORTAL_URL}/agents/{id}`): a public listing goes through staff
  review. Don't flip it for them.

Close on the win and what's next:
> "✅ Your channel is connected. Ask me anytime: *'what parsed from my channel
> today?'*, *'how's my agent doing?'* (portfolio-check), or *'change my fee'*
> (agent-manager)."

---

## Failure and edge handling

- **401 mid-flow** → token expired (OAuth lasts about an hour). Re-run
  `npx @echozero/ez login`, or check the API key, secret and clock (drift over 5
  minutes fails). Resume the exact step.
- **403 on bot-metadata, verify or oauth-url** → not a registered developer, or
  missing `write:agents` / `read:agents` scope. Register (Step 0) or re-login with
  the scopes from echozero-agent-setup Step 1.
- **404** → check the path against `https://docs.echozero.app/llms.txt` and the
  spec, retry once, then name the failed call and stop.
- **Create 400** → name over 20 characters, or `HYPERLIQUID` without
  `signalExecutionDefaults.marketType: "perpetual"`. Fix and retry once.
- **They aren't an admin of the channel** → only an admin can add the bot. Ask
  them to send the Step 3 link and instructions to an admin, then verify once the
  admin confirms.
- **The deep link or invite page doesn't load** → use the manual Telegram path in
  Step 3, or finish in the Dev Portal at `{ECHOZERO_PORTAL_URL}/agents/{id}`.
- **They ask you to add the bot for them** → explain you can't: Telegram and
  Discord require the admin to do it in their own app or browser. Everything else
  you can do.

---

## What else they can do

- **echozero-agent-manager**: pause or resume, change fees, gating, claim fees.
- **echozero-portfolio-check**: P&L, trades, subscribers.
- **echozero-agent-setup**: sign-in, and webhook agents for their own bot or code.

---

## Endpoint / tool cheat sheet (verify against the API spec)

| Purpose | REST call | MCP tool |
|---|---|---|
| Verify auth | `GET {ECHOZERO_MCP_URL}/status` | `get_current_user` |
| Register as developer | `POST {ECHOZERO_MCP_URL}/api/developers/register` | none |
| List my agents | `GET {BASE}/agents` | `get_my_agents` |
| Create signal-group agent | `POST {BASE}/agents` (`signalSourceKind: "signal_group"`) | `create_agent` |
| Update agent (fee, rule, gating) | `PATCH {BASE}/agents/:id` | `update_agent` |
| Telegram bot username | `GET {BASE}/agents/telegram/bot-metadata` | none |
| Discord bot metadata (generic invite) | `GET {BASE}/agents/discord/bot-metadata` | none |
| Discord per-agent invite URL | `GET {BASE}/agents/:id/discord/oauth-url` | none |
| Verify Telegram channel | `POST {BASE}/agents/:id/telegram/verify` | none |
| Verify Discord channel | `POST {BASE}/agents/:id/discord/verify` | none |
| Simulate a call (sandbox) | `POST {ECHOZERO_MCP_URL}/api/sandbox/agents/:id/signals/simulate` | none |
| Sandbox history | `GET {ECHOZERO_MCP_URL}/api/sandbox/agents/:id/signals` | none |
| Suggest a parsing rule | `POST {BASE}/developers/agents/:id/signal-corpus/analyze` | none |
| Ingest log (parsed / skipped) | `GET {BASE}/developers/agents/:id/signals/ingest-log` | none |

Auth: `Authorization: Bearer <oauth-token>` **or** `x-api-key: $ECHOZERO_API_KEY`
plus `x-timestamp` and `x-signature` (Appendix E1 of echozero-agent-setup).

---

## Runner variants

One skill, three runners. The only difference is the sign-in trigger:
- **Claude Code**: `npx @echozero/ez login --client claude` (MCP auto-wired into Claude's config).
- **Codex**: `npx @echozero/ez login --client codex` (writes Codex MCP config).
- **OpenClaw**: `npx @echozero/ez login --client openclaw` (writes OpenClaw config).
Flow, copy, and API contracts are identical across all three. Headless mode
(`ECHOZERO_API_KEY`) is identical too. Don't fork this file per runner; if the
installer needs per-runner artifacts, generate them from this source. (See
CONTRIBUTING.md.)
