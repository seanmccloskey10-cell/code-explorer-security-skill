# Code Explorer Security Skill

A Claude Code skill that audits vibe-coded applications for security vulnerabilities.

Built for the **Next.js + Supabase + Stripe** stack. Focuses on the real mistakes AI-generated code makes.

Part of the [Code Explorer](https://whop.com/code-explorer/) course by Sean McCloskey.

## Install

**Claude Code (global — works in any project):**
```bash
npx skills add https://github.com/seanmccloskey/code-explorer-security-skill --skill vibe-security
```

**Manual install:**
Copy the `vibe-security/` folder to `~/.claude/skills/`

## Usage

```
/vibe-security              — full audit
/vibe-security quick        — top 20 checks only
/vibe-security deep         — full audit + advanced patterns
/vibe-security focus:auth   — authentication only
/vibe-security focus:payments — Stripe only
/vibe-security focus:database — Supabase only
```

Or just ask naturally:
- "Is this code secure?"
- "Check my code for security issues"
- "Audit this before I deploy"

## What it checks

| Category | Key issues |
|---|---|
| Secrets & Keys | Hardcoded keys, NEXT_PUBLIC_ mistakes, git history leaks |
| Authentication | Middleware bypass CVE, JWT handling, IDOR |
| Database | Supabase RLS, service_role key exposure, storage buckets |
| Payments | Stripe webhook verification, client-side price trust |
| Next.js | Known CVEs, Server Actions, data leakage to client |
| Rate Limiting | Missing limits on AI/auth/email endpoints |
| Deployment | Debug mode, source maps, security headers |
| Mobile | Expo bundle exposure, secure token storage |
| AI Integration | API key exposure, prompt injection, spending caps |
| Data Access | SQL injection, XSS, mass assignment, input validation |

## Severity guide
- 🔴 Critical — fix immediately, do not deploy
- 🟡 Important — fix before sharing publicly
- 🟢 Good practice — worth doing when you have time

## License
MIT
