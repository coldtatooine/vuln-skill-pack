---
name: supabase-security
description: Use when reviewing, building, or shipping a Supabase-backed app. Footguns: RLS off or too permissive, service_role in the client, weak policies, storage/realtime exposure, trusting client-set columns.
version: 1.0.0
---

# Supabase Security Footguns

Apply when the app uses Supabase (Postgres, Auth, Storage, Edge Functions). The Supabase client runs in the browser and talks to the database directly, so **RLS is the security boundary** — not your application code. Pair with `/scan`, `/preflight`, `/secrets`.

## Verify before flagging

- RLS **enabled with no policies** = deny-all (app broken, table not open). Do not report as an open table.
- Flag `USING (true)` and auth-only policies without ownership — including `TO authenticated USING (true)` (still IDOR).
- `anon` / publishable keys under public env prefixes are expected; only `service_role` (and JWT secrets) in the client is Critical.
- Table RLS ≠ Storage RLS ≠ Realtime — check each surface separately before claiming "covered by RLS."

## 1. Row Level Security (RLS) — the whole ballgame

- **RLS off = the table is fully readable/writable by anyone with the anon key**, which ships to the browser. Every table exposed via the API MUST have RLS enabled.
- Enabling RLS with **no policy** = deny all (safe but broken). Enabling RLS with a **permissive policy** = the real risk. Read every policy.
- Common broken policy: `USING (true)` — allows all rows. Or a policy that checks authentication (`auth.role() = 'authenticated'` / `TO authenticated`) but **not ownership** → any logged-in user reads every row (IDOR at the DB layer).
- Correct ownership pattern: `USING (auth.uid() = user_id)`. Verify both `USING` (read/existing rows) **and** `WITH CHECK` (insert/update) are set — a missing `WITH CHECK` lets a user write rows they can't read.
- Check policies exist for **all** operations: SELECT, INSERT, UPDATE, DELETE. A table with only a SELECT policy may still be freely deleted.

## 2. The `service_role` key

- `service_role` **bypasses RLS entirely.** It is a full-access admin key.
- It must live **server-side only** — never in client code, never under `NEXT_PUBLIC_*`/`VITE_*`/`EXPOSE_*`, never in the browser bundle. Treat exposure as Critical: rotate immediately.
- Use `anon` key in the browser; `service_role` only in trusted server contexts (Route Handlers, Edge Functions, backend jobs).

## 3. Trusting client-set columns

- The client can set any column not blocked by a policy. Fields like `role`, `is_admin`, `credits`, `tenant_id`, `price` must be protected by `WITH CHECK` or set server-side — never trusted from an insert/update coming through the anon client.
- Privilege escalation: user updates their own `role` column to `admin` because the UPDATE policy only checks `auth.uid() = user_id`.

## 4. Storage buckets & Realtime

- Public buckets serve every object to anyone with the URL. Confirm buckets holding user/private files are **private** with storage RLS policies.
- **Storage policies are separate from table RLS** — check them explicitly.
- **Realtime** authorization is also separate — confirm channel/table subscriptions are not broader than table RLS intent.

## 5. Auth & other

- **Email confirmation / signup**: open signup + auto-confirm can let attackers create accounts freely; confirm it matches intent.
- **`security definer` functions / RPC**: run with the definer's privileges — audit them, they can bypass RLS by design.
- **Postgres extensions & exposed schemas**: don't expose internal schemas via the API.
- **JWT secret**: keep the project JWT secret private; leaking it lets an attacker forge any user's token.

## Quick checklist
- [ ] RLS enabled on every API-exposed table
- [ ] Every policy checks ownership (`auth.uid()`), not just authentication; no `USING (true)` / auth-only IDOR
- [ ] Both `USING` and `WITH CHECK` set for write policies
- [ ] Policies cover SELECT/INSERT/UPDATE/DELETE as needed
- [ ] `service_role` key server-only, never in client bundle
- [ ] Privilege columns (role, credits, tenant_id) not client-writable
- [ ] Private storage buckets have storage RLS; Realtime auth checked separately
