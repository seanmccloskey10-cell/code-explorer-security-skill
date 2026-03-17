# Payment Security (Stripe)

## The three mistakes that cost real money

### 1. Trusting client-submitted prices
Never accept a price or amount from the frontend. Always look it up server-side.

```typescript
// WRONG — user can change the price in browser devtools
const { price } = req.body // attacker sends price: 1
await stripe.checkout.sessions.create({ line_items: [{ price_data: { unit_amount: price } }] })

// RIGHT — look up the price from your database or Stripe
const product = await db.products.findUnique({ where: { id: productId } })
await stripe.checkout.sessions.create({
  line_items: [{ price: product.stripePriceId, quantity: 1 }]
})
```

### 2. Missing webhook signature verification
Without this, anyone can send a fake "payment succeeded" event to your webhook endpoint and get your product for free.

```typescript
// WRONG — no verification
export async function POST(req: Request) {
  const body = await req.json() // parsing destroys the signature
  if (body.type === 'checkout.session.completed') {
    await fulfillOrder(body.data.object)
  }
}

// RIGHT — verify the signature using raw body
export async function POST(req: Request) {
  const body = await req.text() // raw body — must be text, not json
  const signature = req.headers.get('stripe-signature')!

  let event
  try {
    event = stripe.webhooks.constructEvent(body, signature, process.env.STRIPE_WEBHOOK_SECRET!)
  } catch {
    return new Response('Invalid signature', { status: 400 })
  }

  if (event.type === 'checkout.session.completed') {
    await fulfillOrder(event.data.object)
  }
}
```

### 3. Trusting client-side payment confirmation
Never trust a redirect or client-side flag to confirm payment. Always verify with Stripe directly.

```typescript
// WRONG — client-side redirect doesn't confirm payment
// /success page: "you're on the success page so you must have paid"

// RIGHT — check payment status server-side
const session = await stripe.checkout.sessions.retrieve(sessionId)
if (session.payment_status === 'paid') {
  await fulfillOrder(session)
}
```

## What to scan for

**Critical:**
- Price or amount values accepted from request body
- Webhook endpoint doesn't call `stripe.webhooks.constructEvent()`
- Stripe secret key (`sk_live_*`) in client-side code or `NEXT_PUBLIC_` prefix
- Test keys (`sk_test_`) used in production environment
- Subscription status checked from client-side state, not verified server-side

**Important:**
- No idempotency keys on payment operations — duplicate charges possible
- No logging of payment events — no audit trail for disputes
- Coupon/discount codes not validated server-side
- Checkout session metadata set from client, not server
- Refund/cancellation endpoints don't require authentication

## Keep test and live keys completely separate
```bash
# .env.local (development)
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_test_...

# Production environment (Vercel/Netlify dashboard)
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_live_...
```
