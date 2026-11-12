# EchoZero Skills

Conversational skills that let a developer run their entire EchoZero account from
inside their AI assistant (Claude Code, Codex, OpenClaw): onboard, ship a trading
agent, send signals, check P&L, and manage fees. The skills talk to the EchoZero
REST API, so they work in any agent runner; when connected over MCP they use the
live tools directly.

> EchoZero is a custodial trading-agent platform, each agent trades on either
> Solana spot (Jupiter) or Hyperliquid perps, while each user
> sees one custodial wallet and one USDC balance. Developers run their strategy code
> off-platform and send **signals** to EchoZero over a webhook; EchoZero executes
> and manages the trades for subscribers.

## Install

This repository is public. Install the skills into your AI assistant with:

```bash
npx skills add EchoZeroApp/skills
```

Then sign in. The command also connects EchoZero's MCP tools to your assistant:

```bash
npx @echozero/ez login --client claude    # or: codex, openclaw
```

(The scoped npm package is `@echozero/ez`; the bare name `ez` belongs to an unrelated
package.) Start with the **agent-setup** skill. It targets the live production API at
`https://mcp.echozero.app`. Full documentation is at
[docs.echozero.app](https://docs.echozero.app).

Headless / CI path: create an API key in the developer portal
(`https://devportal.echozero.app`) and export `ECHOZERO_API_KEY`,
`ECHOZERO_API_SECRET` and `ECHOZERO_API_BASE=https://mcp.echozero.app/api/v1`.
Headless REST calls must be HMAC signed with the API secret; the agent-setup skill
includes a small signing helper in Node and Python.

## The skills

| Skill | What it does |
| --- | --- |
| [`agent-setup`](agent-setup/SKILL.md) | Conversational onboarding: sign in (also connects EchoZero's MCP tools) → create your agent → fire a test signal → optionally scaffold a strategy, set a persona voice, wire execution callbacks → hand off to the portal to go live. |
| [`agent-manager`](agent-manager/SKILL.md) | Manage existing agents: pause/resume, update fees/exits/persona, claim fees, stop-all-trades, go live / take private, archive. |
| [`portfolio-check`](portfolio-check/SKILL.md) | Read-only: agent performance (P&L, win rate, drawdown), trades, subscribers, signal health, wallet balance. |
| [`signal-send`](signal-send/SKILL.md) | Fire a one-off signal — safe sandbox by default, or a live HMAC-signed send (with confirmation). |
| [`market-data`](market-data/SKILL.md) | Solana market data: trending tokens, token price/detail/search, swap quotes. |

## Runners

Each skill is one runner-complete file covering **Claude Code, Codex, and OpenClaw**
inline. The only per-runner difference is the sign-in trigger
(`npx @echozero/ez login --client claude|codex|openclaw`); flow, copy, and API contracts are
identical. We don't fork per-runner files — see the authoring guide, Part 6.

## Contributing / adding a skill

Read **[SKILL-AUTHORING-GUIDE.md](SKILL-AUTHORING-GUIDE.md)** first. It is the quality
playbook: principles, required structure, the EchoZero API reference, a QA checklist,
and the build process. The top rule: **verify every endpoint and field against the
public API spec (<https://mcp.echozero.app/api/docs-json>) and the docs
(<https://docs.echozero.app>) before writing it down. Don't guess.**

## License

See [LICENSE](LICENSE).
