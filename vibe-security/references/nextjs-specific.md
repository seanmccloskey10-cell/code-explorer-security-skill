# Next.js Specific Security

## Known CVEs (patch your Next.js version)

| CVE | Severity | What it does | Fixed in |
|---|---|---|---|
| CVE-2025-29927 | 9.1 | Middleware auth bypass via `x-middleware-subrequest` header | v14.2.25 / v15.2.3 |
| CVE-2025-57822 | 9.1 | SSRF via middleware passing headers into `NextResponse.next()` | v14.2.32 / v15.4.7 |
| CVE-2024-34351 | High | SSRF via Host header in Server Actions with relative redirects | v14.1.1 |

**Check your version:** `cat package.json | grep next`

## NEXT_PUBLIC_ prefix
Any environment variable starting with `NEXT_PUBLIC_` is bundled into the client-side JavaScript. Anyone can read it in browser devtools.

```bash
# Safe — public by design
NEXT_PUBLIC_SUPABASE_URL=https://...
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_...

# NEVER use NEXT_PUBLIC_ for these
NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY=eyJ... # CRITICAL
NEXT_PUBLIC_STRIPE_SECRET_KEY=sk_live_...    # CRITICAL
NEXT_PUBLIC_OPENAI_API_KEY=sk-...            # CRITICAL
```

## Server Actions checklist
Every Server Action is a public POST endpoint. Before any action runs:

```typescript
// Template — apply to every Server Action
'use server'

export async function myAction(input: unknown) {
  // 1. Validate input
  const parsed = schema.safeParse(input)
  if (!parsed.success) throw new Error('Invalid input')

  // 2. Authenticate
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) throw new Error('Unauthorised')

  // 3. Authorise — check ownership, not just login
  const resource = await db.findFirst({ where: { id: parsed.data.id } })
  if (resource.userId !== user.id) throw new Error('Forbidden')

  // 4. Do the thing
}
```

## Data leakage to client components
```typescript
// WRONG — full DB object serialised to client
export default async function Page({ params }) {
  const user = await db.users.findUnique({ where: { id: params.id } })
  return <Profile user={user} /> // sends password hash, email, etc.
}

// RIGHT — select only what the component needs
export default async function Page({ params }) {
  const user = await db.users.findUnique({
    where: { id: params.id },
    select: { name: true, avatar: true } // nothing sensitive
  })
  return <Profile user={user} />
}
```

## What to scan for

**Critical:**
- Next.js version below 14.2.32 or 15.4.7 (SSRF vulnerability)
- `NEXT_PUBLIC_` prefix on secret keys
- API routes with no authentication check
- Server Actions with no authentication or authorisation
- Full database objects passed to Client Components

**Important:**
- `middleware.ts` protecting routes without server-side re-verification
- `getServerSideProps` / `getStaticProps` serialising sensitive data
- Server-only modules imported in client components
- Cache headers missing on sensitive API responses — CDN may serve one user's data to another
