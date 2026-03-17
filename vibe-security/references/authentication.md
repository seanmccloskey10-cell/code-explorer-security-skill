# Authentication & Authorisation

## Critical vulnerabilities

### Next.js middleware bypass (CVE-2025-29927, severity 9.1)
Middleware can be completely bypassed by sending a request with the `x-middleware-subrequest` header. **Never rely on middleware alone for authentication.**

```typescript
// WRONG — middleware-only auth can be bypassed
export function middleware(request: NextRequest) {
  if (!request.cookies.get('token')) {
    return NextResponse.redirect('/login')
  }
}

// RIGHT — re-verify auth inside every Server Action and Route Handler
export async function sensitiveAction() {
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) throw new Error('Unauthorised')
  // now do the thing
}
```

### Server Actions are public POST endpoints
Every Server Action is publicly accessible. Each one needs: (1) input validation, (2) authentication check, (3) authorisation — confirm the user owns the resource, not just that they're logged in.

```typescript
// WRONG
export async function deletePost(postId: string) {
  await db.posts.delete({ where: { id: postId } })
}

// RIGHT
export async function deletePost(postId: string) {
  const user = await getCurrentUser() // throws if not logged in
  const post = await db.posts.findFirst({ where: { id: postId } })
  if (post.userId !== user.id) throw new Error('Forbidden') // ownership check
  await db.posts.delete({ where: { id: postId } })
}
```

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
