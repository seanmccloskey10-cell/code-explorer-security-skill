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

Never include tokens, passwords, or sensitive state in deep link URLs. Deep link parameters are logged by operating systems and appear in analytics tools.

```typescript
// WRONG — token in the URL, logged everywhere
// myapp://reset-password?token=abc123&newPassword=xyz

// RIGHT — short-lived token only, validated server-side
import * as Linking from 'expo-linking'

Linking.addEventListener('url', ({ url }) => {
  const { queryParams } = Linking.parse(url)

  // validate token exists and is a string
  if (!queryParams?.token || typeof queryParams.token !== 'string') return

  // send token to server for validation — never trust client-side
  validateResetToken(queryParams.token)
})
```

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
