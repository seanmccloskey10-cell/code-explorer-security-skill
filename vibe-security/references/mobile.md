# Mobile Security (Expo / React Native)

## The core rule: no secrets in the app bundle
Everything in your JavaScript bundle can be extracted from the compiled app. Anyone can decompile an iOS or Android app and read your code.

## Dangerous prefixes that expose secrets
`EXPO_PUBLIC_` — any variable with this prefix is bundled into the app and readable by anyone.

```bash
# Safe — public by design
EXPO_PUBLIC_SUPABASE_URL=https://...
EXPO_PUBLIC_SUPABASE_ANON_KEY=eyJ...

# NEVER use EXPO_PUBLIC_ for these
EXPO_PUBLIC_OPENAI_API_KEY=sk-...        # CRITICAL — readable from app bundle
EXPO_PUBLIC_STRIPE_SECRET_KEY=sk_live_... # CRITICAL
```

Any API call that requires a secret key must go through a backend proxy — a Route Handler or Edge Function that holds the secret and makes the call on behalf of the app.

## Secure token storage
```typescript
// WRONG — AsyncStorage is unencrypted, readable on rooted devices
import AsyncStorage from '@react-native-async-storage/async-storage'
await AsyncStorage.setItem('token', accessToken)

// RIGHT — use expo-secure-store (uses iOS Keychain / Android Keystore)
import * as SecureStore from 'expo-secure-store'
await SecureStore.setItemAsync('token', accessToken)
```

## Deep link security
```typescript
// WRONG — executing actions directly from deep link params
// myapp://reset-password?token=abc&newPassword=xyz
// attacker crafts a link with a malicious token

// RIGHT — validate and sanitize all deep link params
// never include sensitive data in the URL itself
// use a short-lived token, validate server-side
```

Never include tokens, passwords, or sensitive state in deep link URLs. Deep link parameters are logged by operating systems and can appear in analytics tools.

## Publishing to the App Store (iOS)
Requirements your student's parent should know:
- Apple Developer account: **$99/year**
- Must have a Mac (or use a cloud Mac service like MacStadium)
- App Store review takes 1-3 days, may be rejected
- Age-appropriate content guidelines apply strictly

**Easier alternative for sharing with friends: TestFlight**
- No App Store review
- Friends download via a link, not the App Store
- Free, up to 10,000 testers
- Best path for an 11-year-old's first app

## What to scan for

**Critical:**
- `EXPO_PUBLIC_` prefix on any secret key
- Third-party API calls (OpenAI, Stripe, etc.) made directly from the app without a backend proxy
- Tokens stored in `AsyncStorage` instead of `expo-secure-store`
- API keys in `eas.json` or `app.config.js` that get committed to git

**Important:**
- Deep links passing tokens or sensitive data in the URL
- No certificate pinning for high-security apps (banking, health)
- App requesting excessive permissions (camera, location, contacts) beyond what's needed
