# Code Explorer Skills — Roadmap

## Vision
A suite of 10 Claude Code skills for teaching beginner vibe coders on Preply.
Each skill = one lesson. Run on the student's project. Score out of 10. Fix it live.

## Install (for students)
Ask Claude Code in the chat window:
> "Clone https://github.com/seanmccloskey10-cell/code-explorer-security-skill into .claude/skills/ in my project"

Claude runs the git clone. Student approves with one click. Restart VS Code. Done.

---

## The 10 Skills

| # | Skill | Student hook | Status |
|---|---|---|---|
| 1 | vibe-security | "Is it safe to ship?" | ✅ Done |
| 2 | visual-upgrade | "Does it look like a real product?" | 🔜 Next |
| 3 | launch-checklist | "Am I actually ready to ship?" | Planned |
| 4 | seo-audit | "Can anyone find it on Google?" | Planned |
| 5 | performance-audit | "Is it fast enough?" | Planned |
| 6 | copy-review | "Will people sign up?" | Planned |
| 7 | context-builder | "Why does Claude keep forgetting?" | Planned |
| 8 | error-handling | "What if it breaks at 2am?" | Planned |
| 9 | accessibility-check | "Can everyone actually use it?" | Planned |
| 10 | monetization-check | "How do I make money from this?" | Planned |

### Lesson Arc
```
1. vibe-security      → Is it safe?
2. visual-upgrade     → Does it look real?
3. context-builder    → Does Claude understand my project?
4. launch-checklist   → Am I ready to ship?
5. seo-audit          → Can people find it?
6. copy-review        → Will people sign up?
7. performance-audit  → Is it fast?
8. error-handling     → What if things break?
9. accessibility-check → Can everyone use it?
10. monetization-check → Can I make money?
```

---

## Scan Modes (important for students on $20 plan)

Every skill has two modes:

| Mode | When | Token cost | Use case |
|---|---|---|---|
| `/vibe-security quick` | In the lesson | Very low | Live demo, $20 plan students |
| `/vibe-security` | Homework, pre-deploy | Medium | Full audit, share report next session |
| `/vibe-security deep` | Advanced only | High | Git history scan, advanced patterns |

**Default for lessons: always use `quick`.** Students on the Claude $20 plan
can run quick scans freely without worrying about token cost.
Full scan = homework. Student shares SECURITY_REPORT.md in next lesson.

---

## vibe-security — Upgrade Roadmap

### Current Coverage (12 categories)
- Secrets & API keys
- Supabase RLS
- Authentication (middleware bypass, JWT, IDOR, email enumeration)
- Stripe payments (webhook, server-side price, idempotency)
- Resend email
- Next.js specific (NEXT_PUBLIC_, Server Actions, CVEs)
- WebSockets
- Rate limiting
- Input validation
- Deployment (debug mode, source maps, cron auth, package hallucination)
- Mobile (Expo/React Native)
- AI integrations (prompt injection, spending caps)

### Gaps to Add — Tier 1 (high impact, commonly found)
- **SSRF** — found in every vibe app tested. Next.js has specific CVEs (CVE-2024-34351).
  Check API routes for unvalidated fetch() calls, image remotePatterns wildcards.
- **CORS misconfiguration** — vibe apps ship with `*` and never tighten it.
  Check origin validation, Access-Control-Allow-Credentials + wildcard combination.
- **Security headers** — zero vibe apps set these by default.
  CSP, HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy.
- **NoSQL injection** — MongoDB `$where`/`$ne`/`$regex` operators, Firestore query injection.
- **Race conditions** — concurrent state mutation in payments, coupons, referral claims.

### Gaps to Add — Tier 2 (add after Tier 1)
- Business logic flaws checklist (coupon stacking, negative quantity, price manipulation)
- Dependency confusion / supply chain (private package name collisions)
- Cryptographic weaknesses (MD5/SHA1 passwords, weak RNG, insecure cipher modes)
- Open redirect (URL normalization, path validation bypass)
- Path traversal in file uploads

### Competitive Position
| Tool | Strength | Weakness vs us |
|---|---|---|
| vibeship Scanner | 2000+ patterns, Trivy integration | Not teaching-focused, no plain English, expensive |
| Trail of Bits | Deep crypto/supply chain | Enterprise only, not vibe-coder specific |
| Vibe App Scanner | Lovable/Bolt specific | Surface-level, no fix prompts |
| **vibe-security** | Plain English, fix prompts, stack-specific depth | Missing SSRF, CORS, security headers (for now) |

---

## Hello / Goodbye Skills

The hello/goodbye commands (currently in `.claude/commands/`) should be upgraded
to `.claude/skills/` for richer capabilities:
- Supporting template files (lesson brief, session notes format)
- Dynamic context injection — read git log + SESSION-NOTES.md before Claude starts
- Tool restrictions — read-only access to student files

### /hello skill
Reads student's PRD.md, SESSION-NOTES.md, recent git log.
Briefs the teacher: where the student is, what they struggled with last session,
what to focus on today. Run by teacher before each Preply lesson.

### /goodbye skill
Writes structured session notes to SESSION-NOTES.md.
Documents what was covered, what student struggled with, homework set, next lesson plan.
The next /hello reads this automatically — continuous context across sessions.

---

## Competitor Reference
- **vibeship.co** (Meta Alchemist) — 462 skills, $279 lifetime, enterprise-scale
  Our edge: curated 10, beginner-friendly, teaching context, plain English
- **Anthropic frontend-design skill** — 277K+ installs, confirms visual-upgrade demand
- **scanner.vibeship.co** — web-based fallback for failed lesson installs
