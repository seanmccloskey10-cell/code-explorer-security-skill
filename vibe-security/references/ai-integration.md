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
AI bills can hit thousands overnight if a key leaks or a bot finds your endpoint.

1. Set a hard monthly cap in your OpenAI/Anthropic dashboard
2. Set billing alerts at 50% and 80% of your expected monthly spend
3. Rate limit your AI proxy endpoint and add per-user quotas — see `rate-limiting.md` for implementation

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
