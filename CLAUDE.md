# CLAUDE.md — AIONOS-UNIWEAVE-PLATFORM

Org-level instructions for Claude Code. Every repo in this org inherits these conventions. Repo-level CLAUDE.md files extend or override as needed.

---

## What UniWeave is

UniWeave is a real-time conversational execution platform for enterprise voice agents. We own orchestration, tool execution, latency optimization, and reliability. We integrate with (but don't own) telephony, CRM, and analytics as standalone products.

## Commit conventions

Format: `<type>(<scope>): <description>`

Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `ci`

Examples:
```
feat(chat-engine): add Azure OpenAI streaming endpoint
fix(whatsapp): handle empty message body gracefully
chore(deps): bump express to 4.19
```

Add `[ai-assisted]` at the end when Claude Code or any AI tool helped write the code:
```
feat(pulse): add post-call webhook handler [ai-assisted]
```

## Branch naming

```
feat/short-description
fix/short-description
chore/short-description
docs/short-description
```

Branch from `main`, PR back to `main`. Keep branches short-lived (2-3 days max).

## Pull requests

Every PR uses the org template. Required fields:
- **What changed** — 1-2 sentences
- **Why** — business reason or Trello card reference
- **How to test** — steps a reviewer can follow
- **AI-assisted checkbox** — check if AI helped write code

PRs need 1 approval before merge.

## Code review philosophy

- Review **intent and tests**, not style. Linters handle formatting.
- For AI-assisted code: verify (a) it does what the description says, (b) tests are meaningful, (c) no security issues.
- Leave comments as suggestions, not demands.

## Testing rule

Every new feature needs at minimum:
- One happy-path test
- One error-path test

Tests must pass before merge. No exceptions.

## Security

- **No secrets in code.** Use `.env` files locally, env vars in deployment.
- Commit `.env.example` (variable names + placeholders only), never `.env`.
- 2FA required for all org members.
- If you find a committed secret, report to Harsh or Agam immediately.

## CLAUDE.md in every repo

Every repo must have a `CLAUDE.md` at the root with:
- What the service does (3 sentences)
- How to run locally
- How to test
- Key files (5 most important)
- Environment variables (names only)
- Architecture notes
- Dependencies on other services

Keep it under 100 lines. Update it when things change. PR changes to CLAUDE.md like code.

## Tech stack

Polyglot — Java, Node.js, Python all welcome. No forced migrations. Use what's right for the service.

## Trello

Our task board lives in Trello (aionos.ai workspace, UniWeave board). Reference Trello card IDs in PRs and commits when applicable.
