---
name: stripe-security
description: Use when reviewing, building, or shipping Stripe payment or billing flows. Footguns: unverified webhooks, client-trusted amounts, secret key exposure, missing idempotency, wrong fulfillment event, unfiltered event types.
version: 1.0.0
---

# Stripe Security Footguns

Apply when the app integrates Stripe (Checkout, Payment Intents, Subscriptions, webhooks). Payment code moves money, so bugs are directly monetizable. Pair with `/scan`, `/preflight`, `/secrets`.

## Verify before flagging

- `pk_*` in the client / public env is expected; only flag `sk_*`, `rk_*`, `whsec_*` outside server-only contexts.
- Distinguish **test** vs **live** keys: still flag committed secret keys, but treat live as higher severity.
- Missing idempotency is a finding when fulfillment has side effects (credits, shipping, entitlements) — not every read-only webhook handler.
- `livemode` checks matter when production must ignore test events (or vice versa); don't flag if both modes are intentionally handled.

## 1. Webhook signature verification (Critical)

- **Every webhook endpoint must verify the Stripe signature** using the endpoint's signing secret (`stripe.webhooks.constructEvent(rawBody, sig, secret)`). Without it, anyone can POST a fake `checkout.session.completed` and get free fulfillment.
- **Use the raw request body**, not parsed JSON. Frameworks that auto-parse the body (Express `json()`, Next.js default body parsing) break verification — the signature is computed over raw bytes. Disable body parsing on the webhook route.
- Reject on verification failure. Don't fall through to processing.
- **Allowlist event types** you handle; ignore or reject unexpected types instead of a catch-all switch that fulfills on anything.
- If production should only process live traffic, check `event.livemode` (or equivalent) before fulfilling.

## 2. Never trust amounts or entitlements from the client

- **Price/amount must come from the server**, looked up from your product catalog by product ID — never sent from the browser. A client that posts `amount: 1` to your Payment Intent creation pays 1 cent for a $100 item.
- Same for quantity, currency, discount, and entitlement (which plan/features the user gets). Compute server-side from trusted data.
- With Checkout, prefer server-created `line_items` referencing Stripe Price IDs over client-supplied `price_data`.

## 3. Fulfill on the right event

- Grant access / ship goods on the **verified webhook** (`checkout.session.completed`, `payment_intent.succeeded`, `invoice.paid`) — **not** on the browser redirect to your success URL. The user can navigate to the success URL without paying.
- Check `payment_status === 'paid'` before fulfilling; a completed session is not always a paid one.

## 4. Secret key exposure

- `sk_live_*` / `sk_test_*` / restricted `rk_*` keys are **server-only**. Never in client code, never under a public env prefix. Only `pk_*` publishable keys belong in the browser. Leaked secret key → rotate immediately (Critical).
- Webhook signing secret (`whsec_*`) is also server-only.

## 5. Idempotency & abuse

- Webhooks can be **delivered more than once**. Make fulfillment idempotent (track processed `event.id`) so a retry doesn't double-grant credits or ship twice.
- Rate-limit payment-creation endpoints to prevent card-testing abuse.

## 6. Other

- Don't log full Stripe objects (may contain PII / partial card data). Redact.
- Validate the `customer`/`user` mapping: ensure the authenticated user owns the session/subscription they're acting on (IDOR into someone else's billing).
- Handle disputed/refunded/`subscription.deleted` events to revoke access.

## Quick checklist
- [ ] Webhook verifies signature over the raw body; rejects on failure
- [ ] Handled event types are allowlisted; `livemode` checked when required
- [ ] Amount, currency, quantity, entitlement computed server-side from trusted data
- [ ] Fulfillment happens on verified webhook, not the success redirect; checks `paid`
- [ ] `sk_*` / `whsec_*` server-only; only `pk_*` in the client
- [ ] Webhook processing is idempotent on `event.id`
- [ ] User ownership verified before billing actions
