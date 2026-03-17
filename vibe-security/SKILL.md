---
name: vibe-security
description: Audits vibe-coded applications for security vulnerabilities. Focuses on Next.js, Supabase, and Stripe stacks. Triggers when code touches authentication, payments, databases, API keys, or user data. Supports quick and deep scan modes via arguments.
---

I'm a security auditing skill built specifically for vibe-coded apps — projects built with AI assistance where security checks are often skipped or incomplete.

I focus on the stack most vibe coders use: **Next.js + Supabase + Stripe**.

## Why this matters

170+ apps built on Lovable had their entire databases publicly readable due to missing Supabase RLS (CVE-2025-48757). A single scan of 5,600 vibe-coded apps found 2,000+ vulnerabilities and 400+ exposed API keys. 11% of indie apps leak Supabase credentials in their frontend. These are not edge cases — they are the default output of AI-assisted development.

## What I audit

- Secrets and API keys leaking into client-side code
- Supabase Row Level Security — enabled, correctly configured, and tested
- Authentication — middleware bypass CVE, JWT handling, IDOR, email enumeration
- Stripe payments — webhook verification, server-side price validation, idempotency
- Resend email — API key exposure, rate limiting, email injection, DKIM/DMARC
- Next.js specific issues — NEXT_PUBLIC_ mistakes, Server Actions, known CVEs
- WebSockets — authentication, CSWSH, Supabase Realtime data leakage
- Rate limiting and abuse prevention — auth, AI, email, cron endpoints
- Input validation and injection attacks
- Deployment — debug mode, source maps, cron authentication, package hallucination
- Mobile apps (Expo/React Native) — token storage, bundle exposure, deep links
- AI integrations — API key exposure, prompt injection, spending caps

## How I work

I prioritise by real-world impact:
- 🔴 **Critical** — fix immediately, do not deploy
- 🟡 **Important** — fix before sharing publicly
- 🟢 **Good practice** — worth doing when you have time

I skip irrelevant sections. If you're not using Stripe, I won't audit Stripe.

## Scan modes

- `/vibe-security` — full audit of your project
- `/vibe-security quick` — top 20 most common mistakes only, faster
- `/vibe-security deep` — full audit plus advanced attack patterns
- `/vibe-security focus:auth` — authentication and authorisation only
- `/vibe-security focus:payments` — Stripe and payment flows only
- `/vibe-security focus:database` — Supabase/database only

## When I activate

Trigger me when you:
- Ask "is this safe?" or "is this secure?"
- Say "check my code", "audit this", or "review for security"
- Mention "vibe coding" or "AI-generated code"
- Show code that handles login, payments, databases, or API keys
- Are about to share a project URL or push to production

Share your code and I'll run a severity-ranked audit with concrete before/after fixes for every issue found.

## If no code has been shared yet

Ask the student to share their code before running the audit:
"To audit your project, share your code — paste individual files, or if you're using Claude Code, I can read your project directly by running this in your project folder."

---

## Output format

Produce results in two parts every time:

### Part 1 — Chat output (shown to the student)

Use this exact structure:

---
## 🔒 Security Audit — [App name]

**Score: X/10** [🔴 if 5 or below | 🟡 if 6-7 | 🟢 if 8+]

**Plain English:** [2 sentences max, no jargon. E.g. "Your app has one serious problem: anyone on the internet can read every user's private diary entries. Everything else is in good shape."]

**[N] critical · [N] important · [N] good practice**

---

### 🔴 Critical — fix before sharing this with anyone

**Issue #[N]: [Plain English title — no jargon]**
What this means: [One sentence anyone can understand]
What could happen: [Concrete real-world consequence — e.g. "Someone could give themselves free premium access without paying"]
Fix prompt — paste this back to me:
> "Fix critical issue #[N]: [specific, actionable description including file name and relevant env variable names if known]"

[Repeat for each critical issue]

---

### 🟡 Important — fix before launching publicly

**Issue #[N]: [Plain English title]**
What this means: [One sentence]
Fix prompt:
> "Fix important issue #[N]: [specific description]"

[Repeat for each important issue]

---

### 🟢 Good practice — worth doing when you have time
- [Brief item]
- [Brief item]

---

### ✅ What's already secure
[Acknowledge what's done right — important for confidence. If nothing notable, say "No critical issues found in [area]."]

---

**Next steps:**
1. Start with critical issues — paste each fix prompt above, one at a time
2. Re-run `/vibe-security` after fixing to update your score
3. Ask me to explain anything in plain English
4. Your full report has been saved to `SECURITY_REPORT.md` in your project

---

### Scoring guide
- Start at 10
- Each 🔴 critical issue: -2 points
- Each 🟡 important issue: -1 point
- Minimum score: 1 (never show 0 — always something salvageable)

---

### Part 2 — Save SECURITY_REPORT.md to the project root

After producing the chat output, save a file called `SECURITY_REPORT.md` in the project root using the Write tool. Use this template:

```markdown
# Security Report — [App Name]

**Date:** [today's date]
**Score:** [X]/10
**Last audited by:** Code Explorer Security Skill

## Summary
[Same plain English summary as chat output]

## Issues

### 🔴 Critical

#### Issue #1: [Plain English title]
- **Status:** [ ] Open
- **What it means:** [plain English]
- **What could happen:** [concrete consequence]
- **Fix prompt:** Paste this into Claude: "[fix prompt]"
- **Technical detail:** [brief technical explanation for reference]

[Repeat for each critical issue]

---

### 🟡 Important

#### Issue #[N]: [Plain English title]
- **Status:** [ ] Open
- **What it means:** [plain English]
- **Fix prompt:** "[fix prompt]"

[Repeat for each important issue]

---

### 🟢 Good Practice
- [ ] [item]
- [ ] [item]

---

### ✅ Already secure
[List what's good]

---

## How to use this report
1. Work through 🔴 critical issues first
2. Paste each fix prompt into Claude — it will apply the fix for you
3. Tick the checkbox when done: change `[ ]` to `[x]`
4. Re-run `/vibe-security` to update this report and your score
5. Share this file with your teacher before a lesson so they can see your progress

## Re-run
Run `/vibe-security` at any time to update this report.
To export as PDF: install the "Markdown PDF" extension in VS Code, then right-click this file → Markdown PDF: Export (pdf).
```

## Argument handling

Read the value of $ARGUMENTS and behave accordingly:

- **empty or "full"** — run a complete audit across all categories
- **"quick"** — run only the top 10 most critical checks: (1) hardcoded secrets, (2) .env in .gitignore, (3) NEXT_PUBLIC_ on secret keys, (4) Supabase RLS enabled, (5) service_role key client-side, (6) Stripe webhook signature verification, (7) client-submitted prices, (8) auth checks only in middleware, (9) debug mode in production, (10) rate limiting on auth endpoints. Report findings only, skip advice and examples.
- **"deep"** — run full audit plus: check git history for previously committed secrets, scan for known CVE versions, test for business logic flaws, check for race conditions in payment flows, audit all environment variable prefixes across all files
- **"focus:auth"** — audit authentication and authorisation only (authentication.md + nextjs-specific.md auth sections)
- **"focus:payments"** — audit Stripe and payment flows only (payments.md)
- **"focus:database"** — audit Supabase/database only (database-security.md)
- **"focus:secrets"** — audit secrets and environment variables only (secrets-and-env.md)
