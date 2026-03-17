# Rate Limiting & Abuse Prevention

## AI almost never adds this
Rate limiting is one of the most consistently skipped security controls in AI-generated code. Without it, your endpoints can be spammed, your AI API bills can explode, and bots can brute-force passwords.

## Endpoints that always need rate limiting

- Login and registration
- Password reset / magic link / OTP requests
- AI API calls (OpenAI, Anthropic — these cost money per request)
- Email and SMS sending
- File upload and processing
- Any endpoint that triggers a payment or writes to a database

## Implementation with Upstash Redis (recommended)

```typescript
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '1 m'), // 10 requests per minute
})

export async function POST(req: Request) {
  const ip = req.headers.get('x-forwarded-for') ?? 'anonymous'
  const { success, remaining } = await ratelimit.limit(ip)

  if (!success) {
    return new Response('Too many requests', { status: 429 })
  }

  // handle request
}
```

## Don't store rate limit counters in public Supabase tables
Supabase tables are accessible via the REST API. Users can query and manipulate their own rate limit counters if stored in a public table. Use:
- Upstash Redis (recommended)
- A private Postgres schema (not `public`)
- Middleware-level rate limiting (Vercel, Cloudflare)

## AI spending caps — always set these

Set hard limits at the provider level AND in your own code:

```typescript
// Per-user usage quota in your database
const usage = await db.usage.findFirst({ where: { userId, month: currentMonth } })
if (usage.tokensUsed >= MONTHLY_LIMIT) {
  throw new Error('Monthly limit reached')
}
```

Go to your OpenAI/Anthropic dashboard and set a hard monthly spending cap. Provider-level caps have some lag — your in-code limits are the first line of defence.

## What to scan for

**Critical:**
- No rate limiting on login, registration, or password reset endpoints
- No rate limiting on AI API proxy endpoints
- AI API keys accessible client-side (no proxy = no rate limiting possible)

**Important:**
- Rate limit counters stored in public Supabase tables
- Only IP-based limiting (defeated by VPN rotation) — add per-user limits too
- No billing alerts on cloud or AI providers
- No per-user quotas on AI usage
