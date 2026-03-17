# Deployment & Infrastructure

## Before you make anything public

**Turn off debug mode**
```bash
# .env.production
NODE_ENV=production    # disables stack traces, verbose errors
DEBUG=false
```

**Check your Next.js version**
Several critical CVEs were patched in 2025. Run:
```bash
npm outdated next
```
Update to at least Next.js 14.2.32 or 15.4.7.

**Remove source maps from production**
Source maps expose your original source code to anyone who opens browser devtools.

```javascript
// next.config.js
module.exports = {
  productionBrowserSourceMaps: false, // default is false — make sure it wasn't changed
}
```

## Environment separation
Three tiers — keep them completely separate:

| Tier | Keys | Database |
|---|---|---|
| Development | `sk_test_`, test Supabase | Local or dev DB |
| Preview/Staging | `sk_test_`, staging Supabase | Separate staging DB |
| Production | `sk_live_`, production Supabase | Production DB — no test data |

Never use production keys in development. Never use test keys in production.

## Vercel / Netlify environment variables
Set production secrets in your hosting dashboard, not just in `.env` files. `.env` files are for local development only.

## Security headers
Add these to `next.config.js`:

```javascript
const securityHeaders = [
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'X-Frame-Options', value: 'DENY' },
  { key: 'X-XSS-Protection', value: '1; mode=block' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  {
    key: 'Strict-Transport-Security',
    value: 'max-age=63072000; includeSubDomains; preload'
  },
]

module.exports = {
  async headers() {
    return [{ source: '/(.*)', headers: securityHeaders }]
  },
}
```

## CORS
```typescript
// WRONG — accepts requests from anywhere
response.headers.set('Access-Control-Allow-Origin', '*')

// RIGHT — restrict to your own domain
response.headers.set('Access-Control-Allow-Origin', 'https://yourdomain.com')
```

Never use wildcard CORS (`*`) on authenticated endpoints.

## What to scan for

**Critical:**
- `NODE_ENV` not set to `production` in production
- Debug mode or verbose error logging enabled
- Source maps deployed to production
- `.git` folder accessible publicly
- Database accessible from the public internet (not just from app servers)
- Secrets baked into Docker images instead of using runtime environment variables
- Default admin panel credentials unchanged

**Important:**
- Security headers not configured
- HSTS not enabled
- Mixed HTTP/HTTPS content
- Development dependencies installed in production
- Error pages revealing stack traces, file paths, or technology versions
- Robots.txt exposing sensitive endpoint paths
