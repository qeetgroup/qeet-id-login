# AGENTS.md — qeet-id-login

**The model-neutral instruction file for coding agents.** [CLAUDE.md](CLAUDE.md),
[GEMINI.md](GEMINI.md) and [.github/copilot-instructions.md](.github/copilot-instructions.md) point
here and add nothing architectural.

## What this repository is

The **hosted authentication UI for Qeet ID** — `login.id.qeet.in`. Next.js 16 App Router, React 19
with the React Compiler, Tailwind v4, `@qeetrix/ui`, bun.

> **This is the only host in Qeet where an interactive session is established.** Every other Qeet
> product redirects here over OIDC. That makes a small repository disproportionately important.

Its README states the design directly: *"a thin, stateless front-end: every action calls the Qeet ID
backend API; this app holds no database and no secrets."*

## Context hierarchy

```text
qeet-context (L0)  →  qeet-id-context (L1)  →  qeet-id-server (the contract + all enforcement)
                                            →  qeet-id-login (L2 — this repository)
```

## Read before changing code

| File | For |
|---|---|
| [qeet-repo.yml](qeet-repo.yml) | Machine-readable identity |
| [docs/llm/architecture-map.md](docs/llm/architecture-map.md) | "Where is X?" |
| [docs/llm/context.md](docs/llm/context.md) | How this app works |
| [docs/llm/boundaries.md](docs/llm/boundaries.md) | What it owns and must not touch |
| [docs/llm/workflows.md](docs/llm/workflows.md) | Adding a flow or a page |

Product-level flow narratives: `qeet-id-context/FLOWS/`.

## Rules

### 1. Stay stateless
**No database, no secrets, no server-side session.** Only `NEXT_PUBLIC_*` variables, which are
inlined at build time and therefore public. If a change needs a secret, it belongs in
`qeet-id-server`.

### 2. Never store a token
The session is the backend's **HttpOnly `qe_ls`** cookie. This app cannot read it and must never
try. Do not put a token in `localStorage`, `sessionStorage`, a cookie, or React state.

The pending MFA challenge token is held **in memory only, never in the URL**. Keep it that way.

### 3. CSRF is not optional
Every request sets `credentials: "include"`. Mutations read the `qe_csrf` cookie and echo it as
`X-CSRF-Token`, seeding it with `GET /healthz` if absent. GET sends no CSRF header.
**Never add a skip flag.** The cookie and header names must match `qeet-id-server` and
`qeet-id-react`.

### 4. Never redirect to an unvalidated URL
`safeReturnTo(returnTo)` guards every `window.location.href`. An open redirect on the login host is
a phishing primitive — **always** run a candidate URL through it.

### 5. The admin portal's only credential is the URL token
`/admin-portal/[token]` is deliberately **unauthenticated** — no cookie, no bearer. Whoever holds
the link holds SAML/SCIM configuration access for that tenant, including a call that returns a
plaintext SCIM token.

**Therefore:** never log the token, never put it in a query string, never include it in an error
message, telemetry, or an analytics event.

### 6. Never invent an endpoint
Verify against the real handler in `qeet-id-server`. This app calls `/v1/...` only.

### 7. Per-tenant branding is server-driven
`GET /v1/oauth/login-context?client_id=…` is fetched in a server component per page. Do not hardcode
tenant branding.

## Commands

```bash
bun install
bun run dev -- -p 3003     # 3003 is REQUIRED, see below
bun run build
bun run typecheck
bun run generate:tree
```

**There is no lint, format or test script in this repository.**

> **Run on 3003.** `bun run dev` alone binds 3000. `qeet-id-server`'s `.env.example` sets
> `LOGIN_BASE_URL=http://localhost:3003` and allowlists 3003 for CORS and WebAuthn origins.
> Passkey ceremonies fail on any other port.

## What CI enforces

**Nothing — this repository has no CI workflow.** The only gate is the Vercel build
(`vercel.json` pins `bun install --frozen-lockfile` and `bun run build`). Run `bun run typecheck`
and `bun run build` yourself before pushing.

## Before you finish

```bash
bun run typecheck && bun run build
git diff
```

## Escalate rather than proceed

Storing a token · weakening CSRF · bypassing `safeReturnTo` · logging an admin-portal token ·
adding a server-side secret · calling an endpoint that does not exist · anything requiring another
repository.
