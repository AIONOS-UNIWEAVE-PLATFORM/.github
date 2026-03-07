# UniWeave Engineering Guidelines

**For**: Nomaan, Upender, Himanshu, and all future UniWeave engineers
**From**: Agam (Head of Product)
**Date**: March 5, 2026

---

## 1. What changed

We've created a new GitHub org: **AIONOS-UNIWEAVE-PLATFORM**

This is the permanent home for all UniWeave engineering work. Previously, our code lived across two orgs (UNIWEAVE-AIONOS and Voice-AI-Project) with no shared standards. The new org gives us:

- One place for all repos
- Consistent branch protection and CI
- Shared PR templates and conventions
- AI-native development standards (CLAUDE.md in every repo)

**Structure**: No teams — everyone is an org member with `write` access. Agam is org owner. Simple and flat. We're 4-5 people, not 50.

**What happens to old repos**: They stay where they are during transfer. Once a repo is transferred to the new org, the old location auto-redirects. Nothing breaks. Old orgs become archives — no new repos created there.

---

## 2. How we work

### Branch workflow

```
main (protected) <- PR required
  |-- feat/short-description
  |-- fix/short-description
  |-- chore/short-description
  |-- docs/short-description
```

- **Never push directly to `main`**. Always open a PR.
- Branch from `main`, PR back to `main`.
- Keep branches short-lived — merge within 2-3 days.
- Delete branches after merge.

### Commit conventions

Format: `<type>(<scope>): <description>`

Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `ci`

Examples:
```
feat(chat-engine): add Azure OpenAI streaming endpoint
fix(whatsapp): handle empty message body gracefully
chore(deps): bump express to 4.19
docs(readme): add local setup instructions
```

**AI-assisted work**: Add `[ai-assisted]` at the end of the commit message when Claude Code or any AI tool helped write the code:

```
feat(pulse): add post-call webhook handler [ai-assisted]
```

This is not a mark of shame — it's a signal for reviewers to focus on intent and tests rather than line-by-line style.

### PR process

Every PR uses the org template (auto-loaded). Fill in:

1. **What changed** — 1-2 sentences
2. **Why** — the business reason or ticket reference
3. **How to test** — steps a reviewer can follow
4. **AI-assisted checkbox** — check if AI helped write code
5. **Trello card link** — connect to the board

PRs need **1 approval** before merge. Agam can approve.

### Code review philosophy

- **Review intent and tests**, not style. Linters handle formatting.
- For AI-assisted code: verify (a) it does what the description says, (b) tests are meaningful, (c) no security issues.
- Leave comments as suggestions, not demands. Use GitHub's "suggestion" feature.

---

## 3. CLAUDE.md — your repo's instruction manual

Every repo gets a `CLAUDE.md` file at the root. This is the first thing Claude Code reads when it opens your project. Think of it as onboarding docs that work for both humans AND AI agents.

### Why it matters

- Claude Code reads it automatically — no setup required
- New team members (human or AI) can be productive in minutes
- It's the single source of truth for "how does this service work"
- It prevents Claude from guessing wrong about your codebase

### What goes in yours

See the [CLAUDE.md template](CLAUDE_MD_TEMPLATE.md). Key sections:
- What the service does (3 sentences max)
- How to run locally
- How to test
- Key files (top 5 files and what they do)
- Environment variables
- Architecture notes
- Dependencies on other services

### Rules

- Keep it accurate. If you change how the service runs, update CLAUDE.md.
- PR changes to CLAUDE.md like code — it's a living document.
- Don't put secrets in it.
- Maximum ~100 lines. Be concise.

---

## 4. Your first task — repo onboarding

Each engineer transfers their assigned repo(s) and opens an onboarding PR.

### Step 1: Transfer your repo

Go to your repo > Settings > General > Danger Zone > Transfer repository > Transfer to `AIONOS-UNIWEAVE-PLATFORM`.

You need admin rights on the repo to transfer it. If you don't have them, ask Agam.

### Step 2: Write your CLAUDE.md

Use the [template](CLAUDE_MD_TEMPLATE.md). Fill in every section. If you're not sure about something, put "TBD" — don't leave it blank.

### Step 3: Ensure README.md exists

Every repo needs a README with:
- Service name and one-line description
- Prerequisites (Node version, Python version, etc.)
- Setup instructions (`npm install`, `pip install -r requirements.txt`, etc.)
- How to run locally
- How to run tests
- Environment variables needed (names only — never commit values)

### Step 4: Add .env.example

List every environment variable the service needs, with placeholder values:
```
AZURE_OPENAI_ENDPOINT=https://your-endpoint.openai.azure.com/
AZURE_OPENAI_KEY=your-key-here
MONGODB_URI=mongodb://localhost:27017/your-db
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
```

**Never commit `.env` files.** Only `.env.example`.

### Step 5: Verify .gitignore

Make sure `.gitignore` includes:
```
node_modules/
.env
.env.local
*.log
dist/
build/
__pycache__/
.pytest_cache/
```

### Step 6: Open your onboarding PR

Branch: `chore/onboarding`
PR title: `chore: add CLAUDE.md, README, .env.example for repo onboarding`

This is your first PR in the new org. It sets the foundation for everything else.

### Repo transfer assignments

| Engineer | Repo(s) to transfer | Priority |
|---|---|---|
| **Nomaan** | `intelliconverse-livekit-frontend` | After #94 Chat Engine milestone |
| **Upender** | `analytics-user-mgmt` | After #96 IntelliPulse milestone |
| **Himanshu** | `intelli-converse-tools-service` | After #98 IntelliRAG milestone |
| **Agam** | `voice-prompt-builder`, `intelliconverse-evals` | This week |
| **New** | `ai-gateway` (Agam creates fresh in new org) | When needed |

**Note**: Transfer after your current sprint card milestone — don't interrupt in-progress work.

---

## 5. Repo standards

### Directory conventions

No enforced monorepo structure. Each service owns its layout. But follow these basics:

- `src/` or `app/` — application code
- `test/` or `__tests__/` — test files
- `.github/workflows/` — CI pipelines (use org templates as starting point)
- `CLAUDE.md` — at the root, always
- `README.md` — at the root, always
- `.env.example` — at the root, always
- `.gitignore` — at the root, always

### CI templates

The org provides starter CI workflows:
- **Node.js**: checkout > setup-node > npm ci > lint > test > build
- **Python**: checkout > setup-python > pip install > ruff check > pytest

Copy the relevant template from [workflow-templates/](workflow-templates/) into your repo's `.github/workflows/` and customize.

---

## 6. AI-native development rules

We all use Claude Code. These rules make AI-assisted development transparent and safe.

### The `[ai-assisted]` tag

Add it to any commit where AI wrote or substantially modified the code. This is:
- **Required** — not optional
- **Not a negative signal** — it's a review hint
- **Applied per-commit** — if some commits in a PR are AI-assisted and others aren't, tag only the relevant ones

### Intent-based review

When reviewing AI-assisted PRs:
1. Does the PR description clearly state what changed and why?
2. Do the tests cover the stated behavior?
3. Are there obvious security issues (hardcoded secrets, SQL injection, etc.)?
4. Does the approach make architectural sense?

Don't review AI code line-by-line for style. Review it for correctness and intent.

### Tests as quality gate

- Every new feature needs at minimum: one happy-path test + one error-path test
- Tests must pass before merge
- If you can't write a test for something, explain why in the PR description

---

## 7. Security basics

- **2FA required** — enable two-factor authentication on your GitHub account
- **No secrets in code** — use `.env` files locally, environment variables in deployment. Never commit API keys, tokens, or passwords.
- **`.env` in `.gitignore`** — always. Use `.env.example` for documentation.
- **Report if you see secrets** — if you find a committed secret, tell Agam immediately. We'll rotate the credential and clean git history.

---

## 8. What's NOT changing

- **Tech stack stays polyglot** — Java, Node.js, Python all welcome. No forced migrations.
- **No code refactoring required** — transfer repos as-is. Clean up over time through normal PRs.
- **Old repos archived, not deleted** — history preserved. Redirects work automatically after transfer.
- **Trello stays** — no migration to GitHub Projects.
- **Deployment process stays the same** — Helm charts, same infra. CI is additive, not replacing anything.
- **Your current sprint work continues** — transfer repos AFTER your current milestone, not during.

---

## 9. RepoMapper — Claude Code token optimization

RepoMapper is an MCP server that gives Claude Code a structural map of any codebase using AST parsing. It reduces token usage by ~97% compared to reading full source files — meaning faster, cheaper, more focused AI assistance.

### One-time setup (per machine)

1. Clone RepoMapper to a sibling directory of your repos:
   ```
   git clone https://github.com/pdavis68/RepoMapper.git ../RepoMapper
   ```

2. Install Python dependencies:
   ```
   pip install --user -r ../RepoMapper/requirements.txt
   ```

3. Add to your repo's `.mcp.json` (create the file at repo root if it doesn't exist):
   ```json
   {
     "mcpServers": {
       "repomapper": {
         "type": "stdio",
         "command": "python",
         "args": ["D:/AA/RepoMapper/repomap_server.py"]
       }
     }
   }
   ```
   **Windows**: use `D:/path/to/RepoMapper/repomap_server.py`
   **Mac/Linux**: use `/home/yourname/RepoMapper/repomap_server.py`

   Adjust the path to match where you cloned it.

### Usage rules

- **Map before reading**: when Claude opens an unfamiliar codebase, it calls `repo_map` first to get a structural overview — no speculative file reading
- **Symbol lookup**: use `search_identifiers` to find specific functions or classes before grepping through files
- **Read only what's needed**: after mapping, Claude reads only the specific files relevant to the task

Python 3.12+ works fine (pyproject.toml says 3.13 but 3.12 is supported).

---

## Questions?

Ask Agam. This is a one-time setup — once your repo is transferred and onboarding PR is merged, you're done. Everything after that is normal development.
