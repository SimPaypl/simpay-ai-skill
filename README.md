# simpay-ai-skill

An [Agent Skill](https://docs.simpay.pl) for integrating with the **SimPay.pl** payment
gateway API — for use with any AI coding agent (Claude Code, OpenAI Codex, Cursor, etc.),
not tied to a specific vendor.

Drop `SKILL.md` into your agent's skills directory and it gives your coding assistant
ready-made, accurate context for creating payments, handling IPN webhooks, verifying
signatures, checking statuses, and issuing refunds against the SimPay API — instead of it
guessing field names or hallucinating an auth scheme.

## Install

Copy this repo into your project's skills directory, keeping the folder name so the skill
is discoverable:

```bash
# Claude Code
git clone https://github.com/SimPaypl/simpay-ai-skill .claude/skills/simpay-api-integration
# or .agents/skills/simpay-api-integration for other agents that follow the same convention

# Codex / other agents
git clone https://github.com/SimPaypl/simpay-ai-skill <your-skills-dir>/simpay-api-integration
```

Or via the [skills.sh](https://skills.sh) CLI:

```bash
npx skills add SimPaypl/simpay-ai-skill
```

Any agent that reads Markdown context files can also just be pointed at `SKILL.md` directly.

## What's in it

[`SKILL.md`](./SKILL.md) covers:

- Auth (API key, base URL, required headers)
- Creating a payment (`directChannel`, request/response, fraud-check pitfalls)
- Payment method values, transaction/refund/subscription statuses
- Refunds
- IPN webhooks: payload structure, signature verification (SHA-256, with a PHP example),
  retry behavior, live IP validation
- Brief pointers to BLIK Level 0 / One-Click / Recurrent, DirectBilling (DCB), deferred
  payments (PayPo, Twisto, BLIK Pay Later), and SMS Premium
- Official SDKs
- A pre-launch checklist

Source docs: https://docs.simpay.pl (full index at https://docs.simpay.pl/llms.txt).

## Updating

This skill is hand-maintained against SimPay's public docs. If something's out of date or
missing, open an issue or PR.

## License

MIT — see [LICENSE](./LICENSE).
