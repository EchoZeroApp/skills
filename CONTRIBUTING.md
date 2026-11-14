# Contributing

Thanks for helping improve the EchoZero skills. This page covers how a skill is laid
out and the checks every change must pass.

## The top rule

**Verify every endpoint against the public OpenAPI spec**
(<https://mcp.echozero.app/api/docs-json>) and the docs
(<https://docs.echozero.app>, index at <https://docs.echozero.app/llms.txt>) before
you write it down. Check the exact path (`/api/v1/…`, `/api/…` or the server root),
field names and enum values. If something is not in the spec or the docs, leave it
out or mark it clearly as unconfirmed. Don't guess.

## Skill structure

Each skill is one Markdown file at `<skill-folder>/SKILL.md`, the layout the
`npx skills` installer discovers. Don't rename existing skill folders or their
`name` fields; installs depend on them.

Sections, in this order:

1. **Frontmatter:** `name` (kebab case, `echozero-…`) and a `description` that
   includes the phrases a user would say ("claim my fees", "what's trending").
2. **Title and intro:** what the skill does, in one paragraph.
3. **Prime directive:** the one safety or UX sentence that governs the skill.
4. **Golden rules:** detect before you ask, confirm before anything outward,
   always leave a headless path, voice rules.
5. **Configuration:** `ECHOZERO_MCP_URL` (production `https://mcp.echozero.app`),
   `ECHOZERO_API_BASE` (`{ECHOZERO_MCP_URL}/api/v1`), `ECHOZERO_API_KEY` and
   `ECHOZERO_API_SECRET`. Match the existing skills.
6. **Step 0, authenticate:** `GET {ECHOZERO_MCP_URL}/status`; on 401 send the
   user to echozero-agent-setup.
7. **Operations / flow:** one step at a time, each with the endpoint or MCP tool,
   the confirmation policy and the visible result.
8. **Failure and edge handling:** 401 (re-auth and resume), 404 (check the docs
   index and the spec), plus skill-specific cases in plain language.
9. **What else they can do:** point to the sibling skills.
10. **Endpoint / tool cheat sheet:** one table, REST call plus MCP tool.
11. **Runner variants:** one file covers Claude Code, Codex and OpenClaw. The only
    difference is `npx @echozero/ez login --client claude|codex|openclaw`. Don't fork
    files per runner; generate per-runner artifacts from the single source if an
    installer ever needs them.

Signing helpers live in one place, `agent-setup/SKILL.md` Appendix E. Point to it
(or to `signInboundWebhook` in `@echozero/sdk`) instead of copying a new signer, and
keep every signing description in line with the docs' webhook security page
(<https://docs.echozero.app/guides/webhook-security>).

## QA checklist (every line must pass)

**Accuracy**
- [ ] Every endpoint, field and enum checked against the OpenAPI spec and the docs.
- [ ] No TODO or "confirm path" placeholders left.
- [ ] Correct `/api/v1` vs `/api` vs server root for every path.

**Safety**
- [ ] Sandbox and virtual wallet mode are the default way to test.
- [ ] Anything that moves money, closes trades, publishes or deletes needs an
      explicit yes, with the impact (amount, subscriber count) shown first.
- [ ] Code that sends live signals says it executes for subscribers and keeps a
      confirmation step.

**Experience**
- [ ] Fastest path to a first result; depth offered afterwards.
- [ ] Results summarized, never raw JSON. A visible result at each step.

**Honesty**
- [ ] Promises only what is built; says plainly what is not available.

**Headless**
- [ ] The `ECHOZERO_API_KEY` path is offered wherever OAuth is.

**Writing**
- [ ] Second person, imperative, concise. No "easy", "simply", "just" or "quick".
- [ ] "EchoZero" as one word. No em dashes.

**Consistency**
- [ ] Same structure, configuration block and cheat-sheet format as the other skills.
- [ ] Runner variants covered.

## Pull requests

Open a pull request against `main` with a short description of what changed and
which endpoints you verified.
