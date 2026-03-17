# Data Access & Input Validation

## TypeScript types are not security
TypeScript catches type errors at compile time. At runtime, user input is untyped. A user can send `{ "price": -999 }` and TypeScript won't stop it.

**Always validate external input at runtime** — API routes, Server Actions, webhooks, URL parameters.

```typescript
import { z } from 'zod'

const schema = z.object({
  title: z.string().min(1).max(200),
  price: z.number().positive().max(10000),
  email: z.string().email(),
})

export async function POST(req: Request) {
  const body = await req.json()
  const parsed = schema.safeParse(body)
  if (!parsed.success) {
    return new Response('Invalid input', { status: 400 })
  }
  // safe to use parsed.data
}
```

## SQL injection
Never build database queries by concatenating user input.

```typescript
// WRONG — SQL injection
const users = await db.$queryRawUnsafe(
  `SELECT * FROM users WHERE name = '${userInput}'`
)
// attacker sends: ' OR '1'='1 — returns all users

// RIGHT — parameterized query
const users = await db.$queryRaw`
  SELECT * FROM users WHERE name = ${userInput}
`
```

With Prisma, use the ORM methods — they're safe by default. Only use `$queryRaw` (tagged template) not `$queryRawUnsafe`.

## XSS (Cross-Site Scripting)
If you display user-submitted content on a page, escape it before rendering.

```typescript
// WRONG — renders user input as HTML
<div dangerouslySetInnerHTML={{ __html: userContent }} />

// RIGHT — React escapes by default
<div>{userContent}</div>

// If you must render HTML (rich text editor), use DOMPurify
import DOMPurify from 'dompurify'
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userContent) }} />
```

## Mass assignment
Don't spread request body directly into database operations.

```typescript
// WRONG — user can send { isAdmin: true } and become an admin
await db.users.update({
  where: { id: userId },
  data: { ...req.body } // dangerous
})

// RIGHT — explicitly pick allowed fields
const { name, bio } = req.body
await db.users.update({
  where: { id: userId },
  data: { name, bio } // only these fields
})
```

## What to scan for

**Critical:**
- User input used in raw SQL queries without parameterisation
- `dangerouslySetInnerHTML` with unsanitised user content
- `req.body` spread directly into database update/create operations
- User input passed to `eval()` or `Function()`
- File path construction from user input (path traversal: `../../etc/passwd`)
- User-controlled URLs fetched without validation (SSRF)

**Important:**
- No runtime input validation on API routes or Server Actions (TypeScript alone is not enough)
- No limits on text field length
- File uploads with no type or size validation
- Redirect destinations accepted from user input without whitelist validation
- Form submissions with no CSRF protection
