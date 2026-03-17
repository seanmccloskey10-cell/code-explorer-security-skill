# WebSocket Security (Real-Time Dashboards)

## Authenticate at connection AND per message

WebSockets have no built-in authentication. The HTTP upgrade handshake is the only point where cookies are sent — after that the connection is persistent and unvalidated.

```typescript
// WRONG — no authentication on WebSocket connection
// Anyone who knows the URL can subscribe to your data stream
const wss = new WebSocketServer({ port: 8080 })
wss.on('connection', (ws) => {
  ws.send(JSON.stringify(stockData))
})

// RIGHT — verify JWT at connection time
import { WebSocketServer } from 'ws'
import { verifyToken } from '@/lib/auth'

const wss = new WebSocketServer({ port: 8080 })
wss.on('connection', async (ws, req) => {
  const token = new URL(req.url, 'http://localhost').searchParams.get('token')

  try {
    const user = await verifyToken(token)
    ws.userId = user.id // attach to connection
  } catch {
    ws.close(1008, 'Unauthorised')
    return
  }

  ws.on('message', (msg) => {
    // validate each inbound message
    const data = JSON.parse(msg.toString())
    if (!allowedActions.includes(data.action)) {
      ws.close(1008, 'Forbidden action')
      return
    }
  })
})
```

## Cross-Site WebSocket Hijacking (CSWSH)

Browsers send cookies with WebSocket upgrade requests. A malicious site can open a connection to your WebSocket server and the session cookie rides along — the WebSocket equivalent of CSRF.

```typescript
// RIGHT — validate Origin header server-side
const ALLOWED_ORIGINS = ['https://yourdomain.com', 'https://www.yourdomain.com']

wss.on('connection', (ws, req) => {
  const origin = req.headers.origin

  if (!ALLOWED_ORIGINS.includes(origin)) {
    ws.close(1008, 'Origin not allowed')
    return
  }
})
```

Also set `SameSite=Strict` or `SameSite=Lax` on session cookies to prevent them being sent cross-origin.

## Real-time dashboard data — scope by user

If your dashboard shows user-specific data (portfolio, personal stats), each subscription must be scoped to that user.

```typescript
// WRONG — broadcasts all users' data to all subscribers
ws.send(JSON.stringify(allPortfolios))

// RIGHT — filter by the authenticated user
ws.send(JSON.stringify(portfolios.filter(p => p.userId === ws.userId)))
```

## Validate all inbound WebSocket messages

Client-to-server messages are user input. Apply the same validation as any API endpoint.

```typescript
// WRONG — trusting client message directly
ws.on('message', (msg) => {
  const { ticker } = JSON.parse(msg.toString())
  subscribeToFeed(ticker) // attacker sends malformed ticker or injection payload
})

// RIGHT — whitelist valid values
const VALID_TICKERS = ['BTC', 'ETH', 'SOL', /* ... */]

ws.on('message', (msg) => {
  const { ticker } = JSON.parse(msg.toString())
  if (!VALID_TICKERS.includes(ticker)) return
  subscribeToFeed(ticker)
})
```

## For Supabase Realtime specifically

Supabase Realtime respects RLS — but only if:
1. RLS is enabled on the table
2. The subscription uses an authenticated client (not the anon key on a public table)

```typescript
// WRONG — subscribing with anon key on a table with no RLS
// broadcasts every row change to any connected client
const supabase = createClient(url, process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY)
supabase.channel('all-changes').on('postgres_changes', { event: '*', schema: 'public', table: 'diary_entries' }, handler).subscribe()

// RIGHT — subscribe with the user's session so RLS applies
const { data: { session } } = await supabase.auth.getSession()
// RLS policies on diary_entries table will filter to only this user's rows
supabase.channel('my-entries').on('postgres_changes', {
  event: '*',
  schema: 'public',
  table: 'diary_entries',
  filter: `user_id=eq.${session.user.id}`
}, handler).subscribe()
```

## What to scan for

**Critical:**
- WebSocket endpoints with no authentication on connection
- Supabase Realtime subscriptions on tables without RLS
- No Origin header validation (CSWSH vulnerability)

**Important:**
- Inbound WebSocket messages not validated before processing
- User-specific data broadcast to all subscribers
- Session tokens passed in WebSocket URL query parameters (appear in server logs)
- No token re-validation when long-lived connections expire
