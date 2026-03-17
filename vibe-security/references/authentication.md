# Authentication & Authorisation

## Critical vulnerabilities

> **Next.js middleware bypass and Server Actions** are covered in `nextjs-specific.md`. Read that file alongside this one.

### JWT handling
```typescript
// WRONG — decode doesn't verify the signature
const payload = jwt.decode(token)

// RIGHT — verify checks signature, expiry, issuer
const payload = jwt.verify(token, process.env.JWT_SECRET, {
  algorithms: ['HS256'], // never allow 'none'
  issuer: 'your-app',
})
```

## What to scan for

**Critical:**
- Auth checks only in middleware, not in Route Handlers or Server Actions
- `jwt.decode()` used instead of `jwt.verify()`
- JWT algorithm set to `"none"` or not explicitly specified
- Protected pages/endpoints accessible without login
- Users can access other users' data by changing an ID in the URL (IDOR)
- Admin features accessible to non-admin users
- Login was skipped "to get it working first" and never re-added

**Important:**
- Tokens stored in `localStorage` instead of `HttpOnly` cookies — XSS can steal them
- Logout doesn't invalidate the session server-side — just clearing the browser isn't enough
- Password reset links don't expire or can be reused
- No limit on failed login attempts — allows brute force
- Session ID doesn't change after login — session fixation risk
- Sensitive actions (password change, payment) don't require re-authentication

## Token storage
```typescript
import { cookies } from 'next/headers'

// WRONG — XSS accessible
localStorage.setItem('token', accessToken)

// RIGHT — HttpOnly cookies, inaccessible to JavaScript
cookies().set('token', accessToken, {
  httpOnly: true,
  secure: true,
  sameSite: 'lax',
  maxAge: 60 * 60 * 24 * 7, // 1 week
})
```
