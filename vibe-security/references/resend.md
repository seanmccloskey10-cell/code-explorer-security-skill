# Resend (Email API) Security

## API key must be server-side only

```typescript
// WRONG — key bundled into client JavaScript
// NEXT_PUBLIC_RESEND_API_KEY is readable by anyone in DevTools
const resend = new Resend(process.env.NEXT_PUBLIC_RESEND_API_KEY)

// RIGHT — server-side only, never in a client component
// app/api/send/route.ts
import { Resend } from 'resend'
const resend = new Resend(process.env.RESEND_API_KEY) // no NEXT_PUBLIC_ prefix
```

## Rate limit your send endpoint

Resend enforces 5 requests/second at the platform level. Your own endpoint has no protection by default. Without rate limiting, anyone can spam your contact form, exhaust your quota, or use your endpoint as a spam relay.

```typescript
// app/api/send/route.ts — add rate limiting
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(3, '1 h'), // 3 emails per IP per hour
})

export async function POST(req: Request) {
  const ip = req.headers.get('x-forwarded-for') ?? 'anonymous'
  const { success } = await ratelimit.limit(ip)
  if (!success) return new Response('Too many requests', { status: 429 })

  // send email
}
```

## Email injection — validate all fields server-side

Never pass user-supplied data directly into `to`, `subject`, or `from` fields.

```typescript
// WRONG — open relay, user controls recipient
const { email, message } = await req.json()
await resend.emails.send({
  from: 'noreply@yourdomain.com',
  to: email,         // attacker sends to anyone they want
  subject: message,  // attacker injects newlines to add headers
  html: message,     // attacker injects script tags
})

// RIGHT — hardcode from, validate to, sanitise content
import { z } from 'zod'

const schema = z.object({
  email: z.string().email(),
  message: z.string().min(1).max(1000),
})

const parsed = schema.safeParse(await req.json())
if (!parsed.success) return new Response('Invalid input', { status: 400 })

await resend.emails.send({
  from: 'noreply@yourdomain.com',  // hardcoded — never from user input
  to: 'you@yourdomain.com',         // hardcoded — contact form goes to YOU, not to user-supplied address
  subject: 'New contact form message',
  text: parsed.data.message,         // plain text, not HTML
})
```

## Domain verification (DKIM, DMARC, SPF)

Without these DNS records your emails land in spam and your domain can be spoofed for phishing.

**Required DNS records in Resend dashboard:**
1. DKIM — proves emails are from you
2. SPF — lists servers allowed to send for your domain
3. DMARC — tells receivers what to do with spoofed emails

Check: Resend Dashboard → Domains → your domain → all records showing green.

Do not send production emails from `onboarding@resend.dev` — add your own domain.

## Separate API keys per environment

```bash
# .env.local (development) — use a test/restricted key
RESEND_API_KEY=re_test_...

# Vercel production dashboard
RESEND_API_KEY=re_live_...
```

A compromised development machine should not give access to production email.

## What to scan for

**Critical:**
- `NEXT_PUBLIC_RESEND_API_KEY` or Resend key in any client-side file
- Resend called directly from a `use client` component
- `to` field populated from user-supplied request body without validation

**Important:**
- No rate limiting on the email send endpoint
- HTML content rendered from user input without sanitisation (XSS via email)
- Same API key used across all environments
- Domain DNS records not configured (emails go to spam, domain spoofable)
- No bounce/complaint monitoring — exceeding 4% bounces risks account suspension
