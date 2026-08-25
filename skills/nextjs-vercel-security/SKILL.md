---
name: nextjs-vercel-security
description: Use when reviewing, building, or shipping a Next.js or Vercel project. Footguns: NEXT_PUBLIC secret leakage, unauthenticated Server Actions and Route Handlers, middleware auth bypass, SSRF, edge/CDN caching of auth responses.
version: 1.0.0
---

# Next.js & Vercel Security Footguns

Apply this when the codebase uses Next.js (App or Pages Router) or deploys to Vercel. These are the high-frequency holes; pair with `/scan`, `/preflight`, and `/secrets`.

## Verify before flagging

- `NEXT_PUBLIC_*` with publishable/anon/`pk_` keys is expected — only flag real secrets (`service_role`, `sk_`, DB URLs, JWT secrets).
- Middleware matcher gaps need a **concrete uncovered protected path**, not a vague "matcher looks incomplete."
- A Server Action that verifies session + ownership is not "unauthenticated" just because it is a public POST endpoint by design.
- CDN/edge caching is only a finding when **authenticated or user-specific** responses can be cached and shared.

## 1. Secret leakage across the client/server boundary

- **`NEXT_PUBLIC_*` is public.** Anything with this prefix is inlined into the JS bundle and shipped to the browser. A real secret here = leaked secret. Only publishable/anon keys belong here.
- **Server-only secrets must never be imported into a Client Component.** A module imported by a `"use client"` file (directly or transitively) ships to the browser. Check import chains, not just the file.
- **`server-only` package**: server code that must never reach the client should `import 'server-only'` to fail the build if it does.
- Supabase `service_role`, Stripe `sk_*`, DB URLs → server only.

## 2. Server Actions

- **Every Server Action is a public POST endpoint.** No auth is applied automatically. Each action must independently verify session AND authorization — do not assume it's only callable from your UI.
- **Origin / CSRF posture:** confirm actions check a trusted origin (or equivalent framework protection) and a valid session before mutating state.
- **Validate and authorize arguments.** Args are attacker-controlled. Re-check ownership/tenant on every ID passed in. Don't trust hidden form fields.
- Actions defined inline in a Server Component are still individually invocable.

## 3. Route Handlers (`route.ts`) & API routes

- No built-in auth. Each handler checks session + role itself.
- IDOR: `/api/orders/[id]` must verify the caller owns the order, not just that they're logged in.
- Validate request body/params before use. Don't reflect errors verbatim.

## 4. Middleware auth bypass

- `middleware.ts` `matcher` gaps let routes slip past auth. Verify the matcher actually covers every protected path (static assets, nested routes, `/api`).
- Middleware runs on the edge — don't treat it as the only auth layer. Enforce authz again at the data layer. Middleware is a filter, not the gate.
- Never rely solely on a cookie's presence; verify the session/token signature.

## 5. SSRF & untrusted fetches

- Server-side `fetch()` with a user-supplied URL → SSRF into internal services / cloud metadata (`169.254.169.254`). Allowlist hosts.
- Next.js Image Optimization: `remotePatterns`/`domains` too broad can proxy arbitrary hosts.

## 6. Other

- **Open redirect**: `redirect(userInput)` / `?next=` without validating it's a same-origin relative path.
- **`dangerouslySetInnerHTML`**: XSS if fed unsanitized input.
- **Source maps / `x-powered-by`**: avoid leaking internals in prod.
- **Caching sensitive data**: don't cache authenticated responses at the CDN/edge (`Cache-Control`, `revalidate`) where another user could receive them.

## Quick checklist
- [ ] No real secret under `NEXT_PUBLIC_*` or in a client-imported module
- [ ] Every Server Action verifies session + authorization + argument ownership (and origin/CSRF posture)
- [ ] Every Route Handler checks auth and ownership (no IDOR)
- [ ] Middleware matcher covers all protected paths; authz re-enforced at data layer
- [ ] User-supplied URLs in server fetch/image config are allowlisted
- [ ] Redirects restricted to same-origin relative paths
- [ ] Authenticated/user-specific responses not cached at CDN/edge
