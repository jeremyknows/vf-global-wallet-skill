---
name: vf-global-wallet
description: |
  Integrate VeeFriends Global Wallet into a Next.js app using Privy's cross-app SDK.
  Use when: adding VeeFriends wallet to an existing PrivyAuth setup or when
  setting up PrivyAuth (email + Google) with VeeFriends wallet included.
  Covers Privy v3 setup, PrivyProvider config, cross-app login, wallet operations,
  CSP headers, and deployment to Vercel.
  Reference implementation: https://github.com/jeremyknows/vf-global-wallet
license: MIT
compatibility: Next.js 14+ (App Router), React 18+, TypeScript, @privy-io/react-auth v3.13.x
metadata:
  author: VeeFriends
  version: "1.0.0"
---

# VeeFriends Global Wallet Integration

Build apps that add VeeFriends Global Wallet to your existing PrivyAuth login or set up PrivyAuth with VeeFriends included.

**Architecture:** Your app is a "requester" that delegates VeeFriends wallet auth to VeeFriends (the "provider"). Users sign in via a popup on VeeFriends' domain. All wallet operations (sign, transact) also open consent popups on VeeFriends' domain. Your app never touches private keys.

**Reference implementation:** [vf-global-wallet](https://github.com/jeremyknows/vf-global-wallet) — a minimal working example deployed at https://vf-wallet-test.vercel.app

## What You Get Back from Cross-App Login

Cross-app authentication gives your app **a wallet address — not a full user profile.** This is by design. VeeFriends owns the user relationship; your app just gets proof that someone controls a specific Ethereum wallet.

**Data your app receives:**
- Wallet address (`0x...`) via the cross-app linked account
- Account type (`cross_app`)

**Data your app does NOT receive:**
- Email address
- Display name
- Profile picture
- Any other personal information from the user's VeeFriends account

### Implications for Existing Auth Systems

If your app already has authentication (Google OAuth, email/password, etc.), you need to decide how VeeFriends wallet login fits in:

| Approach | How it works | Best for |
|----------|-------------|----------|
| **Wallet as add-on** | Users sign in with your existing auth, then connect their VeeFriends wallet as a secondary action (like linking a Discord account). Wallet address becomes a profile attribute. | Apps with existing users and auth. Keep your current login flow, add wallet features on top. |
| **Privy for all auth** | Move all authentication to Privy (it supports Google, email, and cross-app). Privy links accounts automatically when the same email appears across methods. | New apps or apps ready to migrate auth. Cleanest unified experience. |
| **Wallet as primary login** | VeeFriends wallet is the only sign-in method. No email auth. | Apps built exclusively for VeeFriends holders. |

**This skill’s default path assumes you already use Privy for auth** and you are adding VeeFriends wallet as a post-login connect flow.

**If you already have auth, "wallet as add-on" is the simplest path.** Keep your existing login flow and add wallet connection as a feature. No auth migration needed.

**If you use Privy for all auth,** Privy can link accounts when the same user signs in via Google on your app and also connects their VeeFriends wallet — but only because Privy sees both methods on *your* app. The email is never shared from VeeFriends' side.

## Prerequisites

1. **Privy Dashboard account** — Create at https://dashboard.privy.io
2. **Requester App ID** — Create a new app in Privy Dashboard (this is YOUR app)
3. **VeeFriends Provider App ID** — `cm5158iom02kdwmj4wj527lc4`
4. **Next.js 14+ project** with App Router, React 18+, TypeScript

### Privy Dashboard Setup (Required)

In your Privy Dashboard app settings:
- Enable "Cross-App Authentication" under Login Methods
- Add VeeFriends as a cross-app provider using Provider App ID `cm5158iom02kdwmj4wj527lc4`
- Enable Email and Google login methods (recommended defaults)
- No server-side API keys needed — both app IDs are public client-side identifiers
  
If the provider app ID ever stops working, re-check Privy Dashboard > Ecosystem > Integrations.

**Default path (recommended):** Add VeeFriends wallet to an existing PrivyAuth setup.
- Keep your current login methods and PrivyProvider.
- Enable cross-app auth + add the VeeFriends provider in Privy Dashboard.
- Add the VeeFriends connect flow post-login (profile/settings) and store the wallet address.

**If you are setting up PrivyAuth for the first time:** Use Email + Google as your login methods, then add the VeeFriends wallet connect flow. VeeFriends wallet is not a replacement for email/Google.

## Branding: VeeFriends Wallet Icon

**Required.** When building a VeeFriends wallet login experience, you **must** use the official VeeFriends logo for the wallet icon. The logo is included in this skill at `assets/vf-logo.svg`.

Copy it into your project's public directory:

```bash
cp ~/.claude/skills/vf-global-wallet/assets/vf-logo.svg public/vf-logo.svg
```

Then use it wherever you display the VeeFriends wallet button or branding:

```tsx
import Image from 'next/image';

<Image src="/vf-logo.svg" alt="VeeFriends" width={80} height={80} priority />
```

**Do not** substitute a generic wallet icon, a placeholder, or an AI-generated logo. The official VeeFriends cat logo in the white circle is the required brand mark for all VeeFriends wallet integrations.

The SVG is 800x800 with a white circle background — it renders well on both light and dark backgrounds at any size.

## Step 1: Install Dependencies

```bash
npm install @privy-io/react-auth@3.13.1
```

Pin the exact version (no `^` or `~`). Privy SDK has breaking changes between versions. Review changelog before updating.

## Step 2: Environment Variables

Create `.env.local`:

```env
NEXT_PUBLIC_PRIVY_APP_ID=your_requester_privy_app_id
NEXT_PUBLIC_VF_PROVIDER_APP_ID=cm5158iom02kdwmj4wj527lc4
```

> The VeeFriends Provider App ID is `cm5158iom02kdwmj4wj527lc4`.

Both are public client-side IDs (not secrets). The `NEXT_PUBLIC_` prefix is required — Next.js bakes these into the client bundle at build time. **You must redeploy after changing them.**

For Vercel: Set both in Dashboard > Settings > Environment Variables.

### Config Validation

Create `src/lib/config.ts` — fail fast if env vars are missing:

```typescript
if (!process.env.NEXT_PUBLIC_PRIVY_APP_ID) {
  throw new Error('Missing NEXT_PUBLIC_PRIVY_APP_ID — check .env.local');
}
if (!process.env.NEXT_PUBLIC_VF_PROVIDER_APP_ID) {
  throw new Error('Missing NEXT_PUBLIC_VF_PROVIDER_APP_ID — check .env.local');
}

export const PRIVY_APP_ID = process.env.NEXT_PUBLIC_PRIVY_APP_ID;
export const VF_PROVIDER_APP_ID = process.env.NEXT_PUBLIC_VF_PROVIDER_APP_ID;

// Chains the VeeFriends wallet supports
export const SUPPORTED_CHAINS = [
  { id: 1, name: 'Ethereum' },
  { id: 8453, name: 'Base' },
] as const;

export const DEFAULT_CHAIN_ID = 1;
```

## Step 3: PrivyProvider Setup

Create `src/app/providers.tsx`:

```tsx
'use client';

import { PrivyProvider } from '@privy-io/react-auth';
import { PRIVY_APP_ID } from '@/lib/config';

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <PrivyProvider
      appId={PRIVY_APP_ID}
      config={{
        appearance: { theme: 'dark' },
        embeddedWallets: {
          ethereum: { createOnLogin: 'off' },
          solana: { createOnLogin: 'off' },
        },
      }}
    >
      {children}
    </PrivyProvider>
  );
}
```

**If you already have a `PrivyProvider`:** merge the `embeddedWallets` config above into your existing config and keep your current login methods. Do not remove existing methods.
  
If only specific routes use Privy, you can scope `Providers` to those route layouts instead of the root layout.

### Critical: Disable Embedded Wallet Creation

`createOnLogin: 'off'` under both `ethereum` and `solana` is **mandatory**. Your app uses the VeeFriends cross-app wallet — you must NOT create separate embedded wallets.

**Privy v3 API note:** `createOnLogin` is nested under `embeddedWallets.ethereum` (not top-level like v2).

## Step 4: Root Layout — Prevent SSG Crash

In `src/app/layout.tsx`:

```tsx
import { Providers } from './providers';

// CRITICAL: Privy crashes during static site generation
export const dynamic = 'force-dynamic';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className="dark">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

**Why `force-dynamic`?** Privy SDK reads app IDs at import time and makes API calls during initialization. During Next.js static site generation (SSG), this crashes silently — the build succeeds but the deployed site shows errors or a blank page. `force-dynamic` forces server-side rendering at request time, avoiding this entirely.

## Step 5: Cross-App Login

```tsx
'use client';

import { useState } from 'react';
import { useCrossAppAccounts, usePrivy } from '@privy-io/react-auth';
import { VF_PROVIDER_APP_ID } from '@/lib/config';

export function ConnectWalletButton() {
  const { ready } = usePrivy();
  const { loginWithCrossAppAccount } = useCrossAppAccounts();
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const handleLogin = async () => {
    setError(null);
    setIsLoading(true);
    try {
      await loginWithCrossAppAccount({ appId: VF_PROVIDER_APP_ID });
    } catch (err) {
      const message = err instanceof Error ? err.message : 'Authentication failed';

      if (message.includes('popup') || message.includes('blocked')) {
        setError('Popup was blocked. Please allow popups for this site and try again.');
      } else if (message.includes('cancelled') || message.includes('denied') || message.includes('closed')) {
        setError('Authentication was cancelled. Please try again.');
      } else if (message.includes('not found') || message.includes('no account')) {
        setError('You need an existing VeeFriends account. Create one at veefriends.com first.');
      } else {
        setError(`Authentication failed: ${message}`);
      }
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <button onClick={handleLogin} disabled={!ready || isLoading}>
      {!ready ? 'Initializing...' : isLoading ? 'Opening VeeFriends...' : 'Connect VeeFriends Wallet'}
    </button>
  );
}
```

**Use this connect flow after your app’s normal login.** It is not intended to replace email/Google auth.
Place the button on a post-login screen (profile, settings, or linked accounts).

### Key Points

- **`ready` gate**: Always check `ready` before calling any Privy method. The SDK needs ~100-500ms to initialize.
- **No account creation**: `loginWithCrossAppAccount()` does NOT create accounts. Users must already have a VeeFriends account. Handle the "not found" error with clear messaging.
- **Popup-based**: Opens a popup on VeeFriends' domain. Users with popup blockers need to allow your site.
- **Programmatic login**: This bypasses Privy's built-in modal UI. You control the entire login experience.

## Step 6: Extract Wallet Address

```typescript
import type { User } from '@privy-io/react-auth';

const ETH_ADDRESS_RE = /^0x[a-fA-F0-9]{40}$/;

export function getWalletAddress(user: User | null): string | undefined {
  const crossAppAccount = user?.linkedAccounts?.find(
    (account) => account.type === 'cross_app'
  );
  const address =
    crossAppAccount?.type === 'cross_app'
      ? crossAppAccount.embeddedWallets?.[0]?.address
      : undefined;

  return address && ETH_ADDRESS_RE.test(address) ? address : undefined;
}
```

### SDK Quirk: camelCase vs snake_case

Privy's API returns `embedded_wallets` (snake_case), but the SDK transforms it to `embeddedWallets` (camelCase) at runtime. **Use camelCase in your code.** The TypeScript types may show snake_case — ignore them, camelCase works at runtime.

## Step 7: Wallet Operations

All wallet operations use `useCrossAppAccounts()` (not `usePrivy()`). Every method requires `{ address }` as the second argument.

**Single-action lock is mandatory.** Only one Privy popup can be open at a time. Calling multiple methods concurrently causes "in-flight transaction limit" errors.

```tsx
const { signMessage, signTypedData, sendTransaction } = useCrossAppAccounts();
const [activeAction, setActiveAction] = useState<string | null>(null);

const address = getWalletAddress(user);
const isDisabled = activeAction !== null;
```

### Sign Message

```tsx
const handleSignMessage = async () => {
  setActiveAction('signMessage');
  try {
    const signature = await signMessage('Hello from my app', { address });
    console.log('Signature:', signature);
  } catch (err) {
    console.error('Sign failed:', err);
  } finally {
    setActiveAction(null);
  }
};
```

- No gas cost. Opens popup for user consent.
- Returns hex signature string.
- If you use signatures to verify ownership, verify the signature server-side before storing the wallet address.

### Sign Typed Data (EIP-712)

```tsx
const handleSignTypedData = async () => {
  setActiveAction('signTypedData');
  try {
    const typedData = {
      domain: { name: 'My App', version: '1', chainId: 1 },
      types: {
        MyMessage: [
          { name: 'content', type: 'string' },
          { name: 'timestamp', type: 'uint256' },
        ],
      },
      primaryType: 'MyMessage' as const,
      message: {
        content: 'Hello from my app',
        timestamp: BigInt(Math.floor(Date.now() / 1000)),
      },
    };
    const signature = await signTypedData(typedData, { address });
    console.log('Typed signature:', signature);
  } catch (err) {
    console.error('Sign failed:', err);
  } finally {
    setActiveAction(null);
  }
};
```

- Use native `BigInt` for numeric types (not strings or hex).
- `primaryType` needs `as const` for TypeScript.
- No gas cost. Opens popup for user consent.

### Send Transaction

```tsx
const handleSendTransaction = async () => {
  setActiveAction('sendTransaction');
  try {
    const txHash = await sendTransaction(
      {
        to: recipientAddress as `0x${string}`,
        value: BigInt(0),
        data: '0x' as `0x${string}`,
        chainId: 1,
      },
      { address },
    );
    console.log('TX hash:', txHash);
  } catch (err) {
    console.error('Transaction failed:', err);
  } finally {
    setActiveAction(null);
  }
};
```

- **Requires gas fees** even for 0 ETH transfers (~$0.50-2 on Ethereum, ~$0.01 on Base).
- Returns transaction hash (not receipt). Use an RPC provider or block explorer to check confirmation.
- Opens popup for user consent showing transaction details.

## Step 8: Authentication Guard

Protect routes that require authentication:

```tsx
'use client';

import { usePrivy } from '@privy-io/react-auth';
import { useRouter } from 'next/navigation';
import { useEffect } from 'react';

export function AuthGuard({ children }: { children: React.ReactNode }) {
  const { ready, authenticated } = usePrivy();
  const router = useRouter();

  useEffect(() => {
    if (ready && !authenticated) {
      router.replace('/');
    }
  }, [ready, authenticated, router]);

  if (!ready) return <div>Loading...</div>;
  if (!authenticated) return null;
  return <>{children}</>;
}
```

Use `router.replace('/')` (not `push`) so the back button doesn't return to an authenticated page after logout.

### Sign Out

```tsx
const { logout } = usePrivy();

const handleSignOut = async () => {
  await logout();
  router.replace('/');
};
```

**Important:** `logout()` only clears the local session. The cross-app link between your app and VeeFriends persists — it cannot be removed via the Privy API. Users signing back in will automatically reconnect.

## Step 9: Security Headers

Add to `next.config.ts`:

```typescript
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
          { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
          {
            key: 'Content-Security-Policy',
            value: [
              "default-src 'self'",
              "script-src 'self' 'unsafe-inline' 'unsafe-eval' https://*.privy.io",
              "connect-src 'self' https://*.privy.io https://auth.privy.io https://*.infura.io https://*.alchemy.com https://*.walletconnect.com https://*.walletconnect.org wss://*.walletconnect.com wss://*.walletconnect.org",
              "frame-src https://*.privy.io https://auth.privy.io",
              "style-src 'self' 'unsafe-inline'",
              "img-src 'self' data: https:",
              "font-src 'self' data:",
              "base-uri 'self'",
              "form-action 'self'",
            ].join('; '),
          },
        ],
      },
    ];
  },
};

export default nextConfig;
```

### Why These CSP Domains?

| Domain | Purpose |
|--------|---------|
| `*.privy.io` / `auth.privy.io` | Privy SDK API calls + OAuth popups |
| `*.infura.io` / `*.alchemy.com` | Ethereum RPC providers Privy uses internally |
| `*.walletconnect.com` | WalletConnect protocol (Privy dependency) |

`unsafe-inline` and `unsafe-eval` are required by the Privy SDK and Next.js. For hardened production apps, consider nonce-based CSP.

## Account Funding (Gas Sponsorship)

For production apps, Privy offers **gas sponsorship** so users don't need to hold ETH for transaction fees. This is configured in the Privy Dashboard under your app settings — no code changes needed.

When enabled, Privy's infrastructure pays gas fees on behalf of your users. This is especially useful for:
- Onboarding users who don't own crypto yet
- Low-value transactions where gas costs exceed the transaction value
- Base chain transactions (already ~$0.01, but sponsorship removes the friction entirely)

Contact VeeFriends or check Privy Dashboard > Account Funding for setup details.

## Common Gotchas

### 1. "Embedded wallets" — Privy runtime uses camelCase
TypeScript types may show `embedded_wallets`. At runtime, use `embeddedWallets`. This is a confirmed SDK behavior, not a bug.

### 2. SSG causes blank pages
If your deployed site shows a blank page but builds successfully, you're missing `export const dynamic = 'force-dynamic'` on layouts that render Privy components.

### 3. Multiple popups crash
Only one Privy popup can be active at a time. Use an `activeAction` lock state to disable all action buttons while one operation is in progress.

### 4. Users must have a VeeFriends account
`loginWithCrossAppAccount()` does NOT create accounts. First-time users get an error. Direct them to veefriends.com to create an account first.

### 5. Logout is local only
`logout()` clears your app's session. The cross-app link persists on VeeFriends' side. This is by design — cross-app links cannot be revoked by requester apps.

### 6. Transactions need gas even for 0 ETH
Warn users about gas costs. Ethereum mainnet: ~$0.50-2. Base: ~$0.01. Consider using Account Funding (gas sponsorship) to cover this.

### 7. React StrictMode double-mounts
React 18+ StrictMode runs effects twice in development. Use `useRef` guards for one-time operations:

```tsx
const hasRun = useRef(false);
useEffect(() => {
  if (!hasRun.current) {
    // one-time logic here
    hasRun.current = true;
  }
}, []);
```

## Error Boundary (Recommended)

Wrap your app in an ErrorBoundary to catch Privy SDK crashes:

```tsx
'use client';

import { Component, type ReactNode } from 'react';

export class ErrorBoundary extends Component<
  { children: ReactNode },
  { hasError: boolean; error: Error | null }
> {
  constructor(props: { children: ReactNode }) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  render() {
    if (this.state.hasError) {
      return (
        <div style={{ padding: '2rem', textAlign: 'center' }}>
          <h1>Something went wrong</h1>
          <p>Try refreshing the page.</p>
          {this.state.error && <pre>{this.state.error.message}</pre>}
          <button onClick={() => window.location.reload()}>Refresh</button>
        </div>
      );
    }
    return this.props.children;
  }
}
```

Use in layout: `<ErrorBoundary><Providers>{children}</Providers></ErrorBoundary>`

## Quick Reference

| What | How |
|------|-----|
| Wallet icon | Copy `assets/vf-logo.svg` to `public/` — **required branding** |
| Initialize | `<PrivyProvider appId={PRIVY_APP_ID} config={...}>` |
| Login | `loginWithCrossAppAccount({ appId: VF_PROVIDER_APP_ID })` |
| Check ready | `const { ready, authenticated } = usePrivy()` |
| Get address | `getWalletAddress(user)` — see Step 6 for null-safe extraction |
| Sign message | `signMessage(message, { address })` |
| Sign typed data | `signTypedData(typedData, { address })` |
| Send transaction | `sendTransaction({ to, value, chainId }, { address })` |
| Logout | `logout()` — local only, cross-app link persists |
| Prevent SSG crash | `export const dynamic = 'force-dynamic'` |
| Disable embedded wallets | `embeddedWallets: { ethereum: { createOnLogin: 'off' } }` |

**Optional: Apple Sign In**  
Add Apple if your product is iOS-heavy or already uses it. Otherwise it adds UI/config overhead. If you include Apple, enable it in the Privy Dashboard and keep VeeFriends as a separate connect flow.

## Supported Chains

| Chain | ID | Gas Cost |
|-------|----|----------|
| Ethereum Mainnet | 1 | ~$0.50-2 per tx |
| Base | 8453 | ~$0.01 per tx |

## Tech Stack of Reference Implementation

- Next.js 16 (App Router), React 19, TypeScript 5 — compatible with Next.js 14+ and React 18+
- Privy React SDK v3.13.1 (`@privy-io/react-auth`)
- Tailwind CSS v4 + shadcn/ui (optional — any styling works)
- Deployed on Vercel
