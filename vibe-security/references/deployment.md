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

## Vercel cron job authentication

Vercel cron jobs call a URL via HTTP. Without a secret token check, the route is publicly callable by anyone.

```typescript
// vercel.json
{
  "crons": [{ "path": "/api/send-digest", "schedule": "0 8 * * *" }]
}

// WRONG — publicly triggerable by anyone
export async function GET() {
  await sendDailyDigest()
  return new Response('done')
}

// RIGHT — verify the cron secret
export async function GET(req: Request) {
  const authHeader = req.headers.get('authorization')
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return new Response('Unauthorised', { status: 401 })
  }
  await sendDailyDigest()
  return new Response('done')
}
```

Set `CRON_SECRET` as a random string in your Vercel dashboard environment variables.

## Package hallucination — supply chain risk

AI assistants sometimes reference npm packages that don't exist. Attackers register packages with those hallucinated names. When you run `npm install`, you install the attacker's package.

```bash
# Before installing any AI-suggested package, verify it exists and is legitimate
npm info package-name  # check it exists
# Check: download count, last published date, GitHub repo linked, maintainer history

# Run after every install
npm audit
```

Red flags: package published very recently, zero downloads, no GitHub repo, name very similar to a popular package with a typo.

## Preview deployments

Vercel/Netlify create a public URL for every branch push. If that preview uses production secrets or connects to the production database, anyone with the URL has production access.

- Scope environment variables to production only in your hosting dashboard
- Enable deployment protection on preview environments (requires paid plan)
- Never share preview URLs in public channels

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
