# Secrets & Environment Variables

## The core rule
Never put secret keys in your code. Never put them in files that get uploaded to GitHub. Never put them where a browser can see them.

## Framework prefixes that EXPOSE secrets to the browser
| Framework | Dangerous prefix |
|---|---|
| Next.js | `NEXT_PUBLIC_` |
| Vite | `VITE_` |
| Expo/React Native | `EXPO_PUBLIC_` |
| Create React App | `REACT_APP_` |

**Safe to expose (public by design):** Stripe publishable key (`pk_live_*`), Supabase anon key, Firebase client config, public analytics IDs.

**Never expose:** Supabase `service_role` key, Stripe secret key (`sk_live_*`), database connection strings, JWT signing secrets, OpenAI/Anthropic API keys, OAuth client secrets.

## What to scan for

**Critical:**
- Any key or password written directly in code instead of a `.env` file
- `.env` file not listed in `.gitignore`
- Secret keys with `NEXT_PUBLIC_` prefix (exposes them to the browser)
- AWS keys (`AKIA`), GitHub tokens (`ghp_`), Slack tokens (`xoxb-`)
- Stripe secret keys (`sk_live_`, `sk_test_`)
- Supabase `service_role` key anywhere in frontend code
- SSH keys or certificates committed to the repo
- Secrets previously committed to git history (even if since deleted — they're still in the history)

**Important:**
- `console.log` statements printing API keys, tokens, or user data
- Commented-out credentials or test keys left in code
- Placeholder secrets never replaced (`your-api-key-here`, `sk-xxxx`, `replace-me`)
- Config files containing real secrets instead of placeholders
- Secrets passed as URL parameters (they appear in server logs)
- Docker/docker-compose files containing secrets
- `.env.example` containing real values instead of placeholders

## Detection commands
```bash
# Scan for common secret patterns
git ls-files | xargs grep -l "sk_live_\|service_role\|AKIA\|ghp_\|xoxb-"

# Check git history for secrets ever committed
git log --all --full-history -- .env

# Check .gitignore includes .env
cat .gitignore | grep ".env"
```

## The fix
```bash
# .env (never commit this)
STRIPE_SECRET_KEY=sk_live_...
SUPABASE_SERVICE_ROLE_KEY=eyJ...
OPENAI_API_KEY=sk-...

# .gitignore (always commit this)
.env
.env.local
.env.*.local
```
