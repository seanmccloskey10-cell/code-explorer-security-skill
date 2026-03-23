# CORS & Security Headers

## Why vibe apps always get this wrong

CORS and security headers are invisible until something goes wrong. AI tools never set them — they're not part of "making the app work." Every vibe-coded app ships without them. They're not in the tutorial. They're not in the error message. They're in the breach report.

---

## CORS Misconfiguration

CORS tells browsers which other websites are allowed to make requests to your API.

### The dangerous pattern

```js
// WRONG — allows any website to call your API with the user's credentials
app.use(cors({
  origin: '*',
  credentials: true  // this combination is especially dangerous
}))
```

`Access-Control-Allow-Credentials: true` + `Access-Control-Allow-Origin: *` is blocked by browsers but often misconfigured in subtle ways. The real danger is overly permissive origins:

```js
// WRONG — checks if origin "includes" your domain, so attacker.yourdomain.evil.com passes
const origin = req.headers.origin
if (origin.includes('yourdomain.com')) {
  res.setHeader('Access-Control-Allow-Origin', origin)
}
```

### The right pattern

```js
// RIGHT — explicit allowlist only
const allowedOrigins = [
  'https://yourdomain.com',
  'https://www.yourdomain.com'
]

app.use(cors({
  origin: (origin, callback) => {
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true)
    } else {
      callback(new Error('Not allowed by CORS'))
    }
  },
  credentials: true
}))
```

### In Next.js

```js
// next.config.js
async headers() {
  return [
    {
      source: '/api/:path*',
      headers: [
        {
          key: 'Access-Control-Allow-Origin',
          value: 'https://yourdomain.com'  // not *
        }
      ]
    }
  ]
}
```

### What to check
- Is `Access-Control-Allow-Origin` set to `*` on API routes?
- Is `credentials: true` used with a permissive origin?
- Does the origin validation use substring matching instead of exact match?
- Is CORS applied to admin or internal routes?

---

## Security Headers

Security headers are HTTP response headers that tell the browser how to behave. They're the second line of defence after your code. Zero vibe-coded apps set them. One middleware call adds most of them.

### The six headers that matter

**1. Content-Security-Policy (CSP)**
Prevents XSS by telling the browser which scripts are allowed to run.
```
Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted-cdn.com
```
Without this: injected scripts run freely.

**2. Strict-Transport-Security (HSTS)**
Forces HTTPS. Prevents SSL stripping attacks.
```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```
Without this: attackers can downgrade HTTPS to HTTP.

**3. X-Frame-Options**
Prevents your site being embedded in an iframe (clickjacking).
```
X-Frame-Options: DENY
```
Without this: attackers can trick users into clicking things they don't intend to.

**4. X-Content-Type-Options**
Prevents MIME sniffing — browser won't try to guess file types.
```
X-Content-Type-Options: nosniff
```

**5. Referrer-Policy**
Controls how much of your URL is sent to other sites when users click links.
```
Referrer-Policy: strict-origin-when-cross-origin
```

**6. Permissions-Policy**
Restricts browser features (camera, microphone, geolocation) to what you actually need.
```
Permissions-Policy: camera=(), microphone=(), geolocation=(self)
```

### How to add all of them in Next.js (one block)

```js
// next.config.js
async headers() {
  return [
    {
      source: '/(.*)',
      headers: [
        {
          key: 'X-Frame-Options',
          value: 'DENY'
        },
        {
          key: 'X-Content-Type-Options',
          value: 'nosniff'
        },
        {
          key: 'Referrer-Policy',
          value: 'strict-origin-when-cross-origin'
        },
        {
          key: 'Strict-Transport-Security',
          value: 'max-age=31536000; includeSubDomains'
        },
        {
          key: 'Permissions-Policy',
          value: 'camera=(), microphone=(), geolocation=(self)'
        }
        // CSP requires tuning for your specific app — see below
      ]
    }
  ]
}
```

### CSP requires more care
CSP needs to match your app's actual script/style sources. A too-strict CSP breaks the app. A too-loose one is useless. Start with report-only mode:

```
Content-Security-Policy-Report-Only: default-src 'self'; script-src 'self' 'unsafe-inline'
```

Check the browser console for violations, then tighten.

### How to verify
Use [securityheaders.com](https://securityheaders.com) or run:
```bash
curl -I https://yourdomain.com
```
Look for the six header names in the response.
