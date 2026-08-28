# Repository Context — qeet-id-login

**Level:** L2 · **Status:** active · **Evidence state:** verified · **Last verified:** 2026-08-28
**Verification scope:** routes, API client, CSRF behaviour and endpoint list read from source.
Port requirement cross-checked against `qeet-id-server`'s `.env.example`.

## Identity

The **hosted authentication UI for Qeet ID**, served at `login.id.qeet.in`. Next.js 16.2.6 App
Router, React 19 with the React Compiler, Tailwind v4, `@qeetrix/ui ^1.0.2`, bun 1.3.14.
33 source files, ~2.9k LOC.

**Deliberately thin.** Its README: *"a thin, stateless front-end: every action calls the Qeet ID
backend API; this app holds no database and no secrets."*

## Context inheritance

```text
qeet-context (L0)  →  qeet-id-context (L1)  →  qeet-id-server (contract + enforcement)
                                            →  qeet-id-login (L2 — this document)
```

## Why this repository matters more than its size suggests

**It is the only host where an interactive Qeet ID session is established.** Every other Qeet
product — and every external relying party — redirects here over OIDC. The organization's
cross-product SSO model depends on that being the single point of interactive authentication.

**Evidence:** `qeet-context/DOMAIN.md`, `qeet-id-context/FLOWS/oauth-oidc.md`.

## Responsibilities

Rendering the interactive flows: password, passkey, social, MFA (TOTP + recovery code), signup with
passkey registration, forgot/reset password, **OAuth consent**, **device-grant user-code entry**, and
the **external admin portal**. Applying per-tenant branding.

## Non-responsibilities

**It decides nothing.** Credential verification, MFA policy, token issuance, consent recording and
session lifetime all belong to `qeet-id-server`. It holds no state, no database, no secret.

**Magic-link consumption is not here** — it lives in `qeet-id-console` at `/_auth/magic`. A
product-surface asymmetry worth knowing before looking for it.

## Architecture

```text
browser
  ↓  credentials: "include"  +  X-CSRF-Token on mutations
src/lib/api.ts          the single client
  ↓
api.id.qeet.in
  ↓
backend sets the HttpOnly qe_ls cookie
```

Server components fetch per-tenant branding per page via
`GET /v1/oauth/login-context?client_id=…` (`src/lib/branding.ts`), so a user sees their own
organization's identity before signing in.

## Session and CSRF

**Evidence:** `src/lib/api.ts`

The file's own header comment states the model:

> *"this app is cookie-based: the backend sets the HttpOnly SSO cookie (`qe_ls`), so every request
> uses `credentials: "include"`. Mutations echo the CSRF double-submit token."*

| Concern | Behaviour |
|---|---|
| Session | Backend-set **HttpOnly `qe_ls`** — this app can never read it |
| CSRF | `qe_csrf` cookie echoed as `X-CSRF-Token` on POST/PATCH/DELETE |
| Seeding | If `qe_csrf` is absent, fire `GET /healthz` with credentials, then retry |
| GET | No CSRF header |
| Errors | `ApiError { status, code, message }` parsed from `{error:{code,message}}` |

The cookie and header names must match `qeet-id-server` **and** `qeet-id-react`, which implements the
identical contract.

**MFA:** `POST /v1/auth/session` may return `mfa_required` with an `mfa_token`. That token is held
**in React memory only and never placed in the URL**, then exchanged at `/v1/auth/session/mfa` with a
`remember` flag.

## The admin portal — the highest-risk surface here

`/admin-portal/[token]` lets an IT administrator at a **customer** configure SAML or SCIM **without
a Qeet ID account**.

```text
tenant admin generates a capability-scoped, time-limited link
   ↓
login.id.qeet.in/admin-portal/{token}
   ↓  no cookie session, no bearer JWT
/v1/admin-portal/{token}/{context,saml,scim,scim/token}
```

**The URL path token is the only credential.** Any leak — a referrer header, a log line, browser
history, a screenshot — grants SAML/SCIM configuration access for that tenant, including
`POST .../scim/token`, which returns a **plaintext SCIM token**.

Backend-side the link is capability-scoped (`saml` or `scim`) and bounded (15 min – 7 days,
24 h default). Client-side, the discipline is: **never log it, never put it in a query string, never
include it in telemetry or an error.**

## Redirect safety

`safeReturnTo(returnTo)` guards every `window.location.href` assignment. **An open redirect on the
login host is a phishing primitive** — it is the one page a user is trained to trust with
credentials. Always route a candidate URL through it.

## Configuration

`NEXT_PUBLIC_API_URL`, default `http://localhost:4001`; production `https://api.id.qeet.in`.
Only `NEXT_PUBLIC_*` variables exist — **they are inlined at build time and are therefore public.**
There is no server-side secret.

## Local development

```bash
bun run dev -- -p 3003
```

**3003 is required, not a preference.** `bun run dev` alone binds 3000. `qeet-id-server`'s
`.env.example` sets `LOGIN_BASE_URL=http://localhost:3003` and allowlists 3003 in `ALLOWED_ORIGINS`
and `WEBAUTHN_RP_ORIGINS`. On any other port, passkey ceremonies fail and OAuth redirects point at
the wrong host.

## Testing and CI

**Neither exists.** No test framework, no test files, no lint or format script, and **no
`.github/` directory at all**. The only gate is the Vercel build.

That is a real risk for the repository that owns interactive authentication. `bun run typecheck` and
`bun run build` are the only checks available; run them.

## Security-critical areas

| Area | Path | Risk | Review |
|---|---|---|---|
| CSRF handling, seeding | `src/lib/api.ts` | **Critical** | Security review |
| Admin-portal token handling | `src/lib/admin-portal.ts` | **Critical** | Security review |
| Redirect validation | `safeReturnTo` usage | **Critical** | Security review |
| MFA token handling | `src/app/login/login-form.tsx` | High | Review — memory only |
| Branding injection | `src/lib/branding.ts` | Medium | Review |

## Known constraints

- **No CI, no tests** — for the interactive-authentication host.
- **Port 3003 is load-bearing** and set only by a README instruction, not by configuration.
- `@qeetrix/ui ^1.0.2` is a **major version behind** npm's `2.0.0`.
- i18n is **English only** — `src/i18n/locales/en/` is the only locale.
- The README documents an `/admin-portal` index route that does not exist.
- `.env.example` is tracked despite `.gitignore` covering `.env*` — it was added before the rule.

## Documentation authority

Source > `qeet-id-server`'s handlers > README. Product-level flow narratives live in
`qeet-id-context/FLOWS/`; this document explains only where they are implemented here.
