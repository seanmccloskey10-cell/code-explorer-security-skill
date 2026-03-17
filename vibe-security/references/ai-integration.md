# AI Integration Security

## API keys must be server-side only
AI API calls are expensive. An exposed key means anyone can run requests billed to you.

```typescript
// WRONG — key in client-side code, exposed in browser
const response = await fetch('https://api.openai.com/v1/chat/completions', {
  headers: { Authorization: `Bearer ${process.env.NEXT_PUBLIC_OPENAI_API_KEY}` }
})

// RIGHT — proxy through a Route Handler
// app/api/chat/route.ts (server-side only)
export async function POST(req: Request) {
  const { message } = await req.json()
  const response = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [{ role: 'user', content: message }],
  })
  return Response.json(response.choices[0].message)
}
```

## Spending caps — set these immediately

A leaked API key or an unprotected endpoint can result in a bill of thousands overnight. A real example: an exposed AWS key with no budget cap resulted in a $30,000 bill from someone running ML training jobs on it.

**It is better for your app to go down than for you to wake up with a $10,000 bill.**

### OpenAI
1. Go to platform.openai.com → Settings → Limits
2. Set a **hard monthly spend limit** (e.g. $5 — raise it deliberately as you grow)
3. Set a **soft limit** at 80% to get an email warning before you hit the hard cap

### Anthropic (Claude)
1. Go to console.anthropic.com → Plans & Billing → Usage Limits
2. Set a monthly spend limit
3. Enable email alerts

### Google (Vertex AI / Gemini)
Google Cloud does not have a simple spend cap — set up a **Budget Alert** in the Billing console and use a Cloud Function to disable billing automatically if the threshold is hit.

### In your own code — per-user quotas
Provider caps are a last resort. Your own code is the first line of defence:

```typescript
// Check before every AI call
const usage = await db.usageLimits.findFirst({
  where: { userId, date: today }
})

if (!usage || usage.requestCount >= USER_DAILY_LIMIT) {
  return new Response('Daily limit reached', { status: 429 })
}

// Increment after the call
await db.usageLimits.upsert({
  where: { userId_date: { userId, date: today } },
  update: { requestCount: { increment: 1 } },
  create: { userId, date: today, requestCount: 1 }
})
```

Store these counters in a server-only table — not on a user-writable Supabase table (see `database-security.md` and `rate-limiting.md`).

Rate limit your AI proxy endpoint — see `rate-limiting.md` for implementation.

## Prompt injection
If user input is included in your AI prompts, attackers can manipulate the AI's behaviour.

```typescript
// WRONG — user controls the system prompt
const prompt = `You are a helpful assistant. ${userMessage}` // user can say "ignore previous instructions"

// RIGHT — keep system and user messages separate
const response = await openai.chat.completions.create({
  messages: [
    { role: 'system', content: 'You are a helpful assistant. Only discuss cooking.' },
    { role: 'user', content: userMessage } // separate, can't override system
  ]
})
```

## LLM output is untrusted
Never render AI output as raw HTML — it can contain script tags (XSS).
Never execute AI output as code.
Always validate AI-returned data against a schema before using it.

## What to scan for

**Critical:**
- AI API keys in client-side code or with `NEXT_PUBLIC_` prefix
- No rate limiting on AI proxy endpoints
- User input concatenated directly into system prompts
- No monthly spending cap set at the provider level

**Important:**
- No per-user usage quotas
- AI output rendered as raw HTML without sanitisation
- AI tool/function calling with unrestricted permissions
- No logging of AI API calls (needed for debugging runaway costs)
- Using AI to construct database queries from user input (SQL injection risk)
