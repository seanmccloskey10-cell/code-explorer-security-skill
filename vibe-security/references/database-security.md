# Database Security (Supabase & Firebase)

## The real RLS problem — it's not just "is it on?"

RLS being enabled is not enough. The most dangerous pattern — and the one AI tools consistently miss — is storing sensitive fields like `subscription_status` or `rate_limit` on the same table users are allowed to write to.

**This is what happened in a real breach:** A developer had RLS correctly configured so users could only read and write their own rows. But `subscription_status` and `rate_limit` were columns on the same user table. An attacker updated their own row to set `subscription_status = 'premium'` and `rate_limit = 10000` — giving themselves free premium access and the ability to hammer the AI endpoint and run up a $10,000 bill. The RLS was technically correct. The data architecture was wrong.

**The rule:** Never store server-controlled values on the same table as user-editable data.

```sql
-- WRONG — user can UPDATE their own row and set subscription_status = 'premium'
-- and rate_limit = 10000, then call your AI endpoint 10,000 times
CREATE TABLE profiles (
  id uuid PRIMARY KEY,
  name text,
  subscription_status text,  -- DANGEROUS on a user-writable table
  rate_limit integer,         -- DANGEROUS on a user-writable table
  ai_generations_used integer
);

-- RIGHT — separate tables with different access levels
CREATE TABLE profiles (
  id uuid PRIMARY KEY,
  name text,
  avatar_url text
  -- only data the user is allowed to change
);

CREATE TABLE subscriptions (
  user_id uuid PRIMARY KEY,
  status text,           -- only writable by your server (service_role)
  rate_limit integer,    -- only writable by your server (service_role)
  plan text
  -- no RLS UPDATE policy for users — server-only writes
);
```

**How to audit this specifically — ask Claude:**
- "Can a user modify their subscription status by updating their own row?"
- "Can a user change their own rate limit?"
- "Is there any column on a user-writable table that affects permissions, billing, or feature access?"
- "Can a user give themselves premium access without paying?"

AI tools will confirm RLS is "correctly configured" without catching this. You must ask the specific questions.

---

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

---

## Layer 2 — The edge cases AI tools never catch

### Multiple RLS policies combine with OR, not AND

Most developers assume adding more policies makes access more restrictive. It doesn't. Postgres combines permissive policies with OR — so one loose policy cancels out a tight one. This is the default behaviour and almost nobody knows it.

```sql
-- You think this is restrictive: only admins OR owner can read
-- It's actually: anyone who is owner OR admin can read — the loose policy wins
CREATE POLICY "Owner access" ON documents
  FOR SELECT USING (auth.uid() = owner_id);

CREATE POLICY "Admin access" ON documents
  FOR SELECT USING (is_admin = true); -- if user can set is_admin = true, they read everything

-- To get AND behaviour, use RESTRICTIVE policies explicitly
CREATE POLICY "Must be owner" ON documents
  AS RESTRICTIVE
  FOR SELECT USING (auth.uid() = owner_id);
```

**How to audit:** Ask Claude: "Are any of my RLS policies PERMISSIVE? Could two policies on the same table combine to give broader access than intended?"

---

### SECURITY DEFINER functions bypass RLS entirely

Postgres functions marked `SECURITY DEFINER` run as the function owner — usually a superuser — not as the calling user. RLS is completely ignored. AI tools generate these constantly without flagging it.

```sql
-- WRONG — this function runs as superuser, RLS on 'orders' is ignored
CREATE OR REPLACE FUNCTION get_user_orders(user_id uuid)
RETURNS TABLE(id uuid, total numeric)
LANGUAGE plpgsql
SECURITY DEFINER  -- THIS bypasses all RLS policies on the orders table
AS $$
BEGIN
  RETURN QUERY SELECT id, total FROM orders WHERE orders.user_id = $1;
END;
$$;

-- RIGHT — use SECURITY INVOKER (or just don't mark it at all, INVOKER is the default)
CREATE OR REPLACE FUNCTION get_user_orders(user_id uuid)
RETURNS TABLE(id uuid, total numeric)
LANGUAGE plpgsql
SECURITY INVOKER  -- runs as the calling user, RLS applies normally
AS $$
BEGIN
  RETURN QUERY SELECT id, total FROM orders WHERE orders.user_id = $1;
END;
$$;
```

**How to audit:** Ask Claude: "Do any of my Postgres functions or Supabase Edge Functions use SECURITY DEFINER? What data do they access?"

---

### Views don't inherit RLS from their underlying tables

A view over a table with RLS runs as the view owner (often superuser) — it bypasses all row policies on the underlying table. Every row becomes visible through the view regardless of what RLS says.

```sql
-- Table has RLS — users can only see their own rows
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Own messages" ON messages USING (auth.uid() = user_id);

-- This view bypasses that RLS entirely — all rows visible to everyone
CREATE VIEW all_messages AS SELECT * FROM messages;
-- Any user querying all_messages sees every message

-- RIGHT — use security_invoker on the view (Postgres 15+ / Supabase support)
CREATE VIEW all_messages
WITH (security_invoker = true)
AS SELECT * FROM messages;
```

**How to audit:** Ask Claude: "Do I have any views over tables with RLS enabled? Are those views marked with security_invoker?"

---

### INSERT without WITH CHECK allows row claiming

An INSERT policy without a `WITH CHECK` clause lets a user insert a row with someone else's `user_id` — claiming ownership of rows that should belong to another account. AI scaffold almost never includes `WITH CHECK`.

```sql
-- WRONG — user can insert a row with any user_id, including another user's
CREATE POLICY "Users insert own rows" ON posts
  FOR INSERT WITH CHECK (true); -- no ownership check at all

-- Also wrong — USING applies to reads, not writes
CREATE POLICY "Users insert own rows" ON posts
  FOR INSERT USING (auth.uid() = user_id); -- USING ignored on INSERT

-- RIGHT — WITH CHECK enforces ownership on write
CREATE POLICY "Users insert own rows" ON posts
  FOR INSERT WITH CHECK (auth.uid() = user_id);
```

**How to audit:** Ask Claude: "Do my INSERT RLS policies have WITH CHECK clauses? Can a user insert a row with a different user_id?"

---

### Aggregate queries leak data even with RLS

RLS prevents users from reading individual rows they don't own. But aggregate functions — `COUNT()`, `SUM()`, `AVG()`, `MAX()` — still work across the whole table. A user can't read other users' orders, but they can run queries that reveal information about them.

```sql
-- User can't read other orders (RLS blocks it)
SELECT * FROM orders WHERE user_id != auth.uid(); -- blocked

-- But they CAN do this — reveals business data about other users
SELECT COUNT(*) FROM orders WHERE total > 10000; -- works, leaks aggregate info
SELECT AVG(total) FROM orders; -- works
SELECT MAX(created_at) FROM orders; -- reveals when last order was placed
```

**Fix:** For sensitive tables, apply RLS policies that filter all queries to the current user's data, including aggregates. Consider whether your API should expose raw table access at all or go through controlled server-side functions.

---

### Soft deletes need RLS treatment too

If you use `deleted_at IS NULL` for soft deletes, your RLS policy still needs to consider deleted rows. By default, a user querying their own rows will also see their own deleted rows — and if the policy is wrong, they might see soft-deleted rows belonging to others.

```sql
-- WRONG — user can query deleted rows from other users if policy only checks user_id
CREATE POLICY "Own rows" ON posts
  USING (auth.uid() = user_id); -- no check on deleted_at

-- A user can do: SELECT * FROM posts WHERE deleted_at IS NOT NULL
-- and see all their own deleted content (usually fine)
-- but if policy has any gap, deleted rows from others may be visible

-- RIGHT — decide explicitly: should users see their own deleted rows?
CREATE POLICY "Own active rows" ON posts
  USING (auth.uid() = user_id AND deleted_at IS NULL);
```

---

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
