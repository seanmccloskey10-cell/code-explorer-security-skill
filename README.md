# Code Explorer Security Skill

This is a security tool that plugs into Claude Code. Once installed, you can run it on any project and it will scan your code for security vulnerabilities — the kind that AI-generated code commonly misses.

Built for the **Next.js + Supabase + Stripe** stack.

Part of the [Code Explorer](https://whop.com/code-explorer/) course by Sean McCloskey.

---

## What does it do?

It has two main jobs:

**1. Quick scan — run this while you're building**
Checks the 10 most critical things (hardcoded API keys, database permissions, payment verification, etc). Fast and low cost. Good habit to run before you share your work with anyone.

```
/vibe-security quick
```

**2. Full audit — run this before you deploy**
Deep scan of your entire project. Reads every file, checks every category, and writes a `SECURITY_REPORT.md` checklist to your project folder that you can tick off.

```
/vibe-security
```

Both modes give you a score out of 10, a plain English summary of what's wrong, and copy-paste fix prompts for each issue.

---

## Install

### The easiest way — let Claude do it for you

No terminal needed. Just open your project in VS Code with Claude Code running, then paste this into the chat:

```
Clone https://github.com/seanmccloskey10-cell/code-explorer-security-skill and copy the vibe-security folder into .claude/skills/ inside my project
```

Claude will propose the commands. Click **approve**. That's it.

**Then close VS Code completely and reopen it.** Claude Code reads skills at startup, so it needs a fresh launch.

Once reopened, run:
```
/vibe-security quick
```

---

### What gets installed

After install you'll see a new folder in your project sidebar:

```
your-project/
└── .claude/
    └── skills/
        └── vibe-security/
            ├── SKILL.md
            └── references/
```

That's the skill — just files. You can open and read them.

---

### If Claude can't run the commands — do it manually

**Windows (Command Prompt):** Search "cmd" in the Start menu, paste this:
```cmd
git clone https://github.com/seanmccloskey10-cell/code-explorer-security-skill && xcopy /E /I /Y "code-explorer-security-skill\vibe-security" "%USERPROFILE%\.claude\skills\vibe-security"
```

**Windows (PowerShell or VS Code terminal):** Run these two lines separately:
```powershell
git clone https://github.com/seanmccloskey10-cell/code-explorer-security-skill
xcopy /E /I /Y "code-explorer-security-skill\vibe-security" "$env:USERPROFILE\.claude\skills\vibe-security"
```

**If git clone says "already exists"** — the repo was downloaded before. Skip the clone, just run the copy line.

**Mac / Linux:**
```bash
git clone https://github.com/seanmccloskey10-cell/code-explorer-security-skill && cp -r code-explorer-security-skill/vibe-security ~/.claude/skills/
```

Close and reopen VS Code, then run `/vibe-security quick`.

---

## Updating

When a new version is released, open Command Prompt and run:

```cmd
cd code-explorer-security-skill && git pull && xcopy /E /I /Y "vibe-security" "%USERPROFILE%\.claude\skills\vibe-security"
```

Or in PowerShell (run each line separately):
```powershell
cd code-explorer-security-skill
git pull
xcopy /E /I /Y "vibe-security" "$env:USERPROFILE\.claude\skills\vibe-security"
```

---

## All scan modes

```
/vibe-security quick          — top 10 checks, low token cost (start here)
/vibe-security                — full audit (run before deploying)
/vibe-security deep           — full audit + advanced patterns
/vibe-security focus:auth     — authentication only
/vibe-security focus:payments — Stripe only
/vibe-security focus:database — Supabase only
```

Or just ask naturally:
- "Is this code secure?"
- "Check my code for security issues"
- "Audit this before I deploy"

---

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
