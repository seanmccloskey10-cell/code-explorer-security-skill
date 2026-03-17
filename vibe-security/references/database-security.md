# Database Security (Supabase & Firebase)

## Supabase — the most common mistakes

### RLS is OFF by default
Every new Supabase table has Row Level Security disabled. This means every row is publicly readable and writable via the API. AI scaffold code almost never enables it.

```sql
-- Check which tables have RLS disabled
SELECT tablename, rowsecurity
FROM pg_tables
WHERE schemaname = 'public';

-- Enable RLS on a table
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;

-- Add a policy — users can only see their own rows
CREATE POLICY "Users see own rows" ON posts
  FOR ALL USING ((SELECT auth.uid()) = user_id);
```

### RLS enabled but no policies = broken silently
If RLS is on but no policies exist, all queries return empty results with no error. The app appears to work but shows nothing.

### Service role key on the client side
The `service_role` key bypasses ALL RLS policies. It's the equivalent of giving every user full database admin access.

```typescript
// WRONG — service_role key in frontend code
const supabase = createClient(url, process.env.NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY)

// RIGHT — anon key on client, service_role only in server-side code
// Client component:
const supabase = createClient(url, process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY)

// Server only (Route Handler / Server Action):
const supabaseAdmin = createClient(url, process.env.SUPABASE_SERVICE_ROLE_KEY)
```

### Storage buckets are public by default
Supabase Storage creates buckets as public. Add access policies.

```sql
-- Restrict storage: users can only access their own files
CREATE POLICY "Users access own files" ON storage.objects
  FOR ALL USING (auth.uid()::text = (storage.foldername(name))[1]);
```

### Never use `user_metadata` in RLS policies

`user_metadata` is writable by the authenticated user themselves. Any policy that reads from it can be self-escalated.

```sql
-- WRONG — user can update their own metadata to set role: 'admin'
CREATE POLICY "Admins only" ON admin_settings
  USING (auth.jwt()->>'user_metadata'->>'role' = 'admin');

-- RIGHT — use a server-controlled table for roles
-- profiles table with is_admin managed only via service_role
CREATE POLICY "Admins only" ON admin_settings
  USING (
    EXISTS (
      SELECT 1 FROM profiles
      WHERE profiles.id = auth.uid() AND profiles.is_admin = true
    )
  );
```

### UPDATE policies without column restrictions = mass assignment

An UPDATE policy that allows `auth.uid() = user_id` with no column restriction lets users update ANY column in their row — including `is_admin`, `credits`, `balance`, or `role`.

```sql
-- WRONG — user can update is_admin on their own row
CREATE POLICY "Users update own row" ON profiles
  FOR UPDATE USING (auth.uid() = id);

-- RIGHT — restrict which columns can be updated
-- Use column-level security or restrict in application code
-- Only allow name, bio, avatar — never is_admin, credits, role
CREATE POLICY "Users update own profile" ON profiles
  FOR UPDATE USING (auth.uid() = id)
  WITH CHECK (auth.uid() = id);
-- Then in your API, explicitly select only safe columns to update
```

### Don't store rate limit counters in public tables
Supabase tables are accessible via the REST API. Users can reset their own rate limit counters if stored in a public table. Use a private schema or Upstash Redis.

## What to scan for

**Critical:**
- RLS disabled on any table containing user data
- `service_role` key in client-side code or prefixed with `NEXT_PUBLIC_`
- Storage buckets with no access policies
- Queries that return other users' data (missing `WHERE user_id = auth.uid()`)
- RLS policies that check `auth.uid()` without wrapping in `SELECT` (performance + security issue)

**Important:**
- RLS enabled but no policies written
- `select *` queries returning columns users shouldn't see
- Rate limit counters in public Supabase tables
- Realtime subscriptions not filtered by user
- Edge Functions that don't verify authentication before processing

## Firebase
```javascript
// WRONG — anyone can read/write
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if true; // never do this
    }
  }
}

// RIGHT — users can only access their own documents
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

Note: Subcollections inherit no security from parent documents. Each subcollection needs its own explicit rules.
