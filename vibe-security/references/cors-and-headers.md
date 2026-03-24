# CORS & Security Headers

## CORS misconfiguration

CORS controls which origins (domains) can make requests to your API. Misconfigured CORS lets any website steal your users' data.

### The dangerous combination
```typescript
// WRONG — wildcard origin + credentials = any site can make authenticated requests as your user
app.use(cors({
  origin: '*',
  credentials: true,
}))

// ALSO WRONG — reflecting the origin header blindly
const origin = req.headers.get('origin')
res.setHeader('Access-Control-Allow-Origin', origin) // trusts any origin

// RIGHT — explicit allowlist only
const allowedOrigins = [
  'https://yourdomain.com',
  process.env.NODE_ENV === 'development' ? 'http://localhost:3000' : null,
].filter(Boolean)

app.use(cors({
  origin: (origin, callback) => {
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true)
    } else {
      callback(new Error('Not allowed by CORS'))
    }
  },
  credentials: true,
}))
```

### Next.js API routes
```typescript
// WRONG — open to any origin
async headers() {
  return [{ source: '/api/:path*', headers: [{ key: 'Access-Control-Allow-Origin', value: '*' }] }]
}

// RIGHT — specific origin only
async headers() {
  return [{
    source: '/api/:path*',
    headers: [{ key: 'Access-Control-Allow-Origin', value: 'https://yourdomain.com' }]
  }]
}
```

### What to scan for
**Critical:**
- Access-Control-Allow-Origin: * on any route that handles authenticated data
- Origin reflected from request headers without validation
- credentials: true paired with a wildcard or dynamic origin

**Important:**
- No CORS policy at all on API routes
- Development origins (localhost) left in production config

---

## Security Headers

Security headers are a second line of defence. They limit what attackers can do even if a vulnerability gets through.

### next.config.js — full header setup
```javascript
const securityHeaders = [
  { key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubDomains; preload' },
  { key: 'X-Frame-Options', value: 'SAMEORIGIN' },
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
  {
    key: 'Content-Security-Policy',
    value: [
      "default-src 'self'",
      "script-src 'self' 'unsafe-inline'",
      "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
      "font-src 'self' https://fonts.gstatic.com",
      "img-src 'self' data: blob: https:",
      "connect-src 'self' https://*.supabase.co wss://*.supabase.co",
    ].join('; ')
  },
]

module.exports = {
  async headers() {
    return [{ source: '/(.*)', headers: securityHeaders }]
  },
}
```

### What each header does — plain English

| Header | What it prevents |
|---|---|
| Strict-Transport-Security | Forces HTTPS — prevents downgrade attacks |
| X-Frame-Options | Prevents your site being embedded in iframes (clickjacking) |
| X-Content-Type-Options | Stops browser guessing file types — prevents MIME attacks |
| Referrer-Policy | Controls what URL is sent when users click external links |
| Content-Security-Policy | Limits where scripts/styles can load from — reduces XSS impact |
| Permissions-Policy | Restricts camera, microphone, geolocation access |

### What to scan for
**Critical:**
- No Content-Security-Policy header
- No X-Frame-Options — app can be embedded in iframes

**Important:**
- No Strict-Transport-Security — users can be downgraded to HTTP
- No X-Content-Type-Options
- CSP uses unsafe-inline without a nonce

**Good practice:**
- Test headers at securityheaders.com after deploying
- Move from unsafe-inline CSP to nonce-based CSP once stable
