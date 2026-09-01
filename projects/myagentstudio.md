# MyAgentStudio — Agent Workbench

**Type:** Personal  
**Status:** Live (beta) — deployed and operated in production  
**Built:** AI-assisted (Claude Code) — solo, architecture and product decisions directed throughout  
**Git:** Public — [github.com/mbmorote/myagentstudio](https://github.com/mbmorote/myagentstudio)

*A guided AI-agent workbench: a structured, always-visible view of the agent sits next to an agent-aware AI chat that proposes changes — nothing is written until you review and approve.*

---

## Overview

Building AI agents — for Claude Code, for Copilot, for any tool that takes a system-prompt file — tends to settle into the same loop: ask a chat to draft or revise an agent, paste the result into a `.md` file, repeat. The AI part works. What's missing is visibility: while you're editing through a chat prompt, you never actually *see* the agent — its sections, its config, what changed.

MyAgentStudio fixes that. Every section of the selected agent (Role, Behavior, Guardrails, Output, and any custom sections) is laid out clearly on screen at all times, with an AI chat panel next to it that edits those specific sections in place. The point isn't chat *or* structure — it's chat *with* structure, so an agent is never edited blind.

Built with Claude Code (AI-assisted development): the first and most consequential decision — the platform, not the `.md` file, is the source of truth — the data model, the import safety rules, the multi-tenant/cost-control layering, and every architectural call were directed by Marco; Claude accelerated the implementation against that direction.

---

## In Production

Self-deployed on AWS EC2 behind Caddy auto-HTTPS, SQLite with a daily backup cron, and uptime monitoring — plus CI/CD via GitHub Actions: a push to `master` automatically tests, builds, and deploys, with a health check against the live URL before the run is called good. This is the differentiator against a repo-only side project: it runs, and it ships like a product.

---

## Architecture

```
┌─────────────┬───────────────────────────┬──────────────┐
│  LIBRARY    │  Structured View          │  Raw │ Share │
│             │  ─────────────────────    │  ┄┄┄┄┄┄┄┄┄┄  │
│  Zara ◄     │  ▸ Role                   │  ---         │
│  Aria       │  ▸ Behavior               │  name: Zara  │
│  dev        │  ▸ Guardrails             │  tools: ...  │
│  qa         │  ▸ Output                 │  ---         │
│  ...        │  ─────────────────────    │  # ROLE      │
│             │  AI CHAT: "tighten her    │  ...         │
│             │  guardrails" → [proposal] │  (foldable,  │
│             │                           │   read-only) │
└─────────────┴───────────────────────────┴──────────────┘
```

A single Next.js app — one deploy unit, frontend and backend together. The frontend is React (App Router) with Tailwind; the backend is the same app's own Route Handlers, the only place the Anthropic API key ever lives. Agent data is stored in SQLite via Drizzle, behind a repository layer that keeps every ownership check in one place — there is no code path that can return another user's agent even by omission.

**Design principles that hold across the whole codebase:**
- **The platform is master** — `.md` files are an export target, not the storage format; nothing is ever read from disk at request time.
- **Lossless round-trip** — import → edit → export never silently drops content; an unrecognized frontmatter key or body section is preserved verbatim.
- **Flag, don't block** — a malformed or unrecognized value is stored as-is and surfaced as a validation flag, never silently corrected.
- **Import is AI-assisted; export is deterministic** — turning messy markdown into structure needs judgment, so an LLM is in the loop for import; turning structure back into markdown needs none, so export is pure code with no AI call.

---

## Technology Stack

| Layer | Choice |
|---|---|
| Frontend | Next.js (App Router) · React · TypeScript · Tailwind CSS |
| Backend | Next.js Route Handlers — same app, no second service |
| Storage | SQLite (`better-sqlite3`) · Drizzle ORM · repository layer |
| AI | Anthropic SDK + a generic OpenAI-compatible provider (NVIDIA NIM verified live), behind one gateway |
| Auth | Custom JWT (`jose`), `bcryptjs`, optional Google OAuth (`arctic`) |
| MCP | `@modelcontextprotocol/sdk` — a second, token-authenticated front door for console/CLI clients |
| Ops | AWS EC2 · Caddy (auto-HTTPS) · pm2 · GitHub Actions CI/CD |

---

## Key Features

- **Lossless import** — Structural mode (AI-assisted, restructures a whole file) and Strict mode (verbatim, classification-only); anything the importer can't confidently place becomes its own visible custom section rather than being dropped.
- **Review-before-apply AI chat** — the chat proposes a change to any part of the agent in one instruction; nothing is written until you see a side-by-side diff and click Apply. One field it can never touch: the agent's `name`.
- **LLM gateway with real cost controls** — every AI call (import + chat, either provider) passes through one function: a dry-run switch that blocks all network traffic while still logging what *would* have run, and a per-user hourly rate cap.
- **Agent sharing** — grant another user live, read-only access to an agent by link or email; their only action is "Copy to me," which forks an independent copy with no ongoing link back.
- **Console access via MCP** — Claude Code (or any MCP client) can list, pull, and — write-gated, off by default — push a user's own agents from the terminal, reusing the exact import pipeline and safety net the browser uses.

---

## Key Achievements

**Provider-agnostic LLM gateway** — a single choke point (`lib/ai/gateway.ts`) enforces dry-run mode and per-user rate limiting ahead of any network call, with two live providers (Anthropic and NVIDIA NIM via a generic OpenAI-compatible adapter) resolved behind one interface. Switching providers is a settings toggle, no restart — verified end-to-end against a real NVIDIA NIM account.

**Lossless AI-assisted import** — Structural import lets an LLM restructure an entire messy document under a prompt-enforced safety model (verbatim movement, no meaning rewrite, no hallucination), backed by a deterministic coverage check that turns any undetected content loss into a non-blocking warning rather than a silent drop.

**Production-hardened CI/CD** — the deploy credential is a dedicated SSH key restricted by a forced command to run exactly one script; the server firewall opens itself to the GitHub runner's IP only for the length of a deploy and closes again even on failure; a failing test suite blocks the deploy job outright via native `needs:` semantics — red means nothing ships.

**MCP write path with no second safety story to maintain** — `push_agent` calls the exact same `upsertAgentFromImport` pipeline the browser's Import dialog uses, inheriting its entire existing safety net (pre/post snapshots, coverage check, revision history) for free instead of a parallel write path that could drift out of sync.

---

## Known Gaps

Honest, tracked, and deliberate — a solo project that ships with known gaps beats a perfect one that doesn't ship:

- No automated coverage on the React component tree yet — business logic (`lib/`, `app/api/`) is thoroughly tested; the component layer is verified by manual browser passes only.
- Chat history lives in browser memory for the current tab — no persistence across reloads yet.
- Group organization (filing an agent into multiple groups) — the data model and API are built; the UI is flag-disabled pending a pre-launch review of a few real gaps that turned up on a closer look.
- Single-process deployment — the login rate limiter and LLM cap are in-memory, not shared across instances (fine for a single EC2 box, a known limitation if that ever changes).

---

## Links

- **Live app:** [myagentstudio.dev](https://myagentstudio.dev) — invite-gated beta
- **Repo:** [github.com/mbmorote/myagentstudio](https://github.com/mbmorote/myagentstudio) — AGPL-3.0, full source
