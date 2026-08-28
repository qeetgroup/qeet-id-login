# GEMINI.md — qeet-id-login

**Read [AGENTS.md](AGENTS.md) first.** It is the model-neutral instruction file and the source of
truth for this repository. This file is a pointer and adds no architecture.

```text
GEMINI.md  →  AGENTS.md  →  docs/llm/*
```

## Canonical context

| File | For |
|---|---|
| [qeet-repo.yml](qeet-repo.yml) | Machine-readable repository identity |
| [AGENTS.md](AGENTS.md) | **Rules, commands, what CI enforces** |
| [docs/llm/context.md](docs/llm/context.md) | How this repository actually works |
| [docs/llm/boundaries.md](docs/llm/boundaries.md) | What it owns, and what it must not touch |
| [docs/llm/workflows.md](docs/llm/workflows.md) | Step-by-step for common changes |
| [docs/llm/architecture-map.md](docs/llm/architecture-map.md) | "Where is X?" — fastest path to a file |

Parent context: **L0** `qeetgroup/qeet-context` · **L1** `qeetgroup/qeet-id-context`.

## What this repository is

The **hosted authentication UI for Qeet ID** (`login.id.qeet.in`). Next.js 16 App Router, React 19, `@qeetrix/ui`, bun. **The only host where an interactive Qeet ID session is established** — every other product redirects here over OIDC. Stateless: no database, no secrets.

## Non-negotiables

1. **Never store a token.** The session is the backend's HttpOnly `qe_ls` cookie — unreadable here, by design.
2. **CSRF is not optional.** `credentials: "include"` everywhere; mutations echo `qe_csrf` as `X-CSRF-Token`. Names must match `qeet-id-server` and `qeet-id-react`.
3. **Never redirect to an unvalidated URL** — always use `safeReturnTo`. An open redirect on the login host is a phishing primitive.
4. **The admin-portal URL token is the entire credential.** Never log it, never put it in a query string, never include it in an error or telemetry.
5. **Keep the MFA challenge token in memory** — never the URL, never storage.
6. **Stay stateless.** No server-side secret; only `NEXT_PUBLIC_*`, which is public by construction.

## Commands

`bun run dev -- -p 3003` · `bun run build` · `bun run typecheck`

That list is complete — **do not invent commands.**

## Before finishing

```bash
bun run typecheck && bun run build
git diff
```
