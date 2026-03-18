---
name: lemonsqueezy-nextjs-debug
description: >
  Complete debugging guide for Lemon Squeezy payment flows integrated with Next.js (App Router) deployed on Vercel.
  Use this skill whenever a user reports issues with: post-payment redirects not working, sessions lost after payment,
  webhook not updating state, users landing on the wrong page after checkout, custom_data not arriving in the webhook,
  or any Lemon Squeezy + Vercel integration behaving unexpectedly. Also use when setting up or reviewing a
  Lemon Squeezy checkout flow from scratch in a Next.js app.
---

# Lemon Squeezy + Next.js + Vercel: Complete Payment Flow Debugging

## How to Debug — Start Here

Work through these steps in order before touching any code.

### Step 1 — Check the Lemon Squeezy dashboard first

Go to **Lemon Squeezy → Webhooks → your webhook → Recent Deliveries**.

For each delivery, check:
- **Status**: did LS consider it successful? Did your endpoint return 200?
- **`meta.event_name`**: should be `order_created` for one-time purchases
- **`meta.custom_data`**: is your session/order ID there? What's the exact key name?
- **`data.attributes.status`**: should be `paid`
- **`data.attributes.test_mode`**: are you accidentally in test mode in production?

Also go to **Orders** and find the order — confirm it exists and is marked as paid.

This tells you whether the problem is on the LS side or your app side.

---

### Step 2 — Check Vercel Logs

Go to **Vercel → your project → Logs**, filter by `/api/webhook`.

For each webhook invocation, check:
- **Status**: 200, 401, 500?
- **External APIs**: does it show outgoing requests? If "No outgoing requests" → your DB/update call is never reached
- **Execution duration**: very fast (< 50ms) with no outgoing requests = code is exiting early (failed signature check, wrong event name, missing custom_data key)
- **Errors or warnings**: any thrown exceptions?

Then filter by `/api/analyze` or whatever route serves the gated content — check what it's returning after payment.

---

### Step 3 — Capture a HAR (Network trace)

Open DevTools → Network tab → record the full flow from upload/start through checkout and back.

Look for:
- After `checkout/fulfilment`, where does the browser navigate next?
- Does `/results/[id]?success=true` (or your equivalent) actually get hit?
- What HTTP status does your results/gated page return? **307 = server-side redirect = content not unlocked yet**
- Is there a 301 redirect swallowing your path? (e.g. `yourdomain.com` → `www.yourdomain.com` dropping `/results/[id]`)

---

## Known Bugs & Root Causes

### 1. `custom_data` key mismatch — camelCase vs snake_case
**Symptom:** Webhook fires and returns 200, "No outgoing requests" in Vercel logs, DB never updated.
**Root cause:** Lemon Squeezy serializes all `custom_data` keys to **snake_case** in the webhook payload, regardless of what you passed when creating the checkout. So `{ sessionId: "abc" }` arrives as `{ session_id: "abc" }`.
**Fix:** Read `event.meta?.custom_data?.session_id` in your webhook handler.
**How to confirm:** Lemon Squeezy → Webhooks → Recent Deliveries → inspect raw payload → `meta.custom_data`.

---

### 2. Session/state lost across Vercel serverless instances
**Symptom:** Session created in one API route is invisible to another. User sees blank/reset state after payment.
**Root cause:** Any in-memory store (`globalThis`, module-level variables, Maps) is process-local. Each Vercel serverless invocation may spin up a fresh instance with empty memory.
**Fix:** Use a persistent store — Supabase, Upstash Redis, PlanetScale, etc. All functions that read/write state must go through the external DB.
**How to confirm:** Add a log in your webhook handler. If it logs but no DB call is made, state is in-memory only.

---

### 3. Post-payment redirect drops the path
**Symptom:** After payment, user lands on the home page (`/`) instead of `/results/[id]` or equivalent.
**Root cause:** Your `redirectUrl` uses a bare domain (e.g. `https://yourdomain.com`) but Cloudflare or Vercel redirects it to `https://www.yourdomain.com` via 301 — and that redirect strips the path.
**Fix:** Set your base URL env var to the exact canonical URL including subdomain, e.g. `https://www.yourdomain.com`.
**How to confirm:** In the HAR, check the request chain after `checkout/fulfilment`. If you see a 301 → landing on `/`, the domain redirect is swallowing the path.

---

### 4. Wrong environment variables in production
**Symptom:** DB calls fail silently. Webhooks return 500. Checkout creation fails.
**Common causes:**
- Env var name in code doesn't match what's set in Vercel (e.g. `SUPABASE_URL` vs `NEXT_PUBLIC_SUPABASE_URL`)
- Using anon key instead of service role key when RLS is enabled
- `LEMON_STORE_ID` or `LEMON_VARIANT_ID` only set for Preview, not Production (or vice versa)
- API keys set for wrong environment scope in Vercel
**Fix:** Go to Vercel → Settings → Environment Variables. Verify every key your app uses exists for the correct environment (Production / Preview / Development). Names are case-sensitive.

---

### 5. Local dev flags left on in production
**Symptom:** Payment check is bypassed, full content shown without paying, or mock data returned instead of real output.
**Root cause:** A flag like `MOCK_PAYMENT=true`, `SKIP_AUTH=true`, or any bypass variable is set in Vercel's Production environment.
**Fix:** Ensure any mock/bypass flags are absent or `false` in Production. They should only be `true` in Development/local.
**How to confirm:** Vercel → Settings → Environment Variables → filter by Production environment.

---

### 6. Webhook signature verification failing silently
**Symptom:** Webhook returns 401 or exits early with no DB call, even though the secret looks correct.
**Common causes:**
- Webhook secret not set or mismatched in Vercel
- Reading the request body twice — the second read gets an empty string
- Using `req.json()` before reading raw body for HMAC — destroys the stream
**Fix:** Always read the raw body once with `req.text()`, compute HMAC, then `JSON.parse()` the same string.

---

### 7. Retry loop on results page times out before webhook fires
**Symptom:** User pays, gets redirected to results, sees loading for a few seconds, then gets sent back to the start.
**Root cause:** Your results page polls the API waiting for `paid: true`, but the webhook from LS arrives after the retry loop gives up.
**Fix:** Increase retry count/delay, or switch to client-side polling that doesn't hard-redirect on first failure. Alternatively, use the LS overlay embed so the user never leaves your page — you control the redirect after confirming payment.

---

## Webhook Handler Reference

```typescript
// app/api/webhook/route.ts
export const runtime = 'nodejs'
import crypto from 'crypto'

export async function POST(req: Request) {
  // 1. Read raw body FIRST — before any other req method
  const rawBody = await req.text()

  // 2. Verify signature
  const signature = req.headers.get('x-signature')
  if (!signature) return Response.json({ error: 'Missing signature' }, { status: 401 })

  const secret = process.env.LEMON_WEBHOOK_SECRET!
  const digest = crypto.createHmac('sha256', secret).update(rawBody).digest('hex')
  const sigBuf = Buffer.from(signature, 'hex')
  const digBuf = Buffer.from(digest, 'hex')
  if (sigBuf.length !== digBuf.length || !crypto.timingSafeEqual(sigBuf, digBuf)) {
    return Response.json({ error: 'Invalid signature' }, { status: 401 })
  }

  // 3. Parse body
  const body = JSON.parse(rawBody) as {
    meta?: {
      event_name?: string
      custom_data?: {
        session_id?: string   // ← always snake_case, regardless of what you passed
        [key: string]: string | undefined
      }
    }
    data?: {
      attributes?: {
        status?: string       // "paid"
        test_mode?: boolean
      }
    }
  }

  // 4. Temporary debug log — remove after confirming it works
  console.log('Webhook received:', JSON.stringify({
    event_name: body.meta?.event_name,
    custom_data: body.meta?.custom_data,
    status: body.data?.attributes?.status,
    test_mode: body.data?.attributes?.test_mode,
  }))

  // 5. Handle the event
  if (body.meta?.event_name === 'order_created') {
    const sessionId = body.meta?.custom_data?.session_id  // snake_case ← critical
    if (typeof sessionId === 'string') {
      await markSessionAsPaid(sessionId)  // your DB update here
    }
  }

  return Response.json({ ok: true })
}
```

---

## Checkout Creation Reference

```typescript
const { data, error } = await createCheckout(
  process.env.LEMON_STORE_ID!,
  process.env.LEMON_VARIANT_ID!,
  {
    checkoutData: {
      custom: {
        session_id: sessionId,  // use snake_case here too, for clarity
      },
    },
    productOptions: {
      // Must use exact canonical URL including www if applicable
      redirectUrl: `${process.env.NEXT_PUBLIC_BASE_URL}/results/${sessionId}?success=true`,
    },
    checkoutOptions: {
      embed: true,  // optional: opens as overlay, user never leaves your page
    },
  }
)
```

---

## Quick DB Checks

Check if your session exists after the user uploads/starts:
```sql
SELECT id, paid, created_at FROM sessions WHERE id = 'your-session-id';
```

Check if it was marked paid after webhook fires:
```sql
SELECT id, paid, updated_at FROM sessions WHERE id = 'your-session-id';
```

Manually trigger a webhook resend to test without paying again:
**Lemon Squeezy → Webhooks → Recent Deliveries → Resend**