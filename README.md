# VeeFriends Global Wallet Skill

An [Agent Skill](https://agentskills.io) that teaches AI coding agents how to integrate **VeeFriends Global Wallet** into Next.js apps using Privy's cross-app SDK.

## What It Does

- Guides setup of Privy cross-app authentication with VeeFriends as the wallet provider
- Provides production-tested code for login, message signing, typed data signing (EIP-712), and transactions
- Documents critical gotchas: SSG crashes, camelCase SDK quirk, single-popup limit, no-account-creation constraint
- Includes security headers (CSP) with all required Privy domains
- Covers deployment to Vercel with environment variable configuration

## Install

### Claude Code

```bash
git clone https://github.com/jeremyknows/vf-global-wallet-skill.git ~/.claude/skills/vf-global-wallet
```

### Cursor

```bash
git clone https://github.com/jeremyknows/vf-global-wallet-skill.git ~/.cursor/skills/vf-global-wallet
```

### Other Agents

Copy `SKILL.md` into your agent's skills directory. See [Agent Skills spec](https://agentskills.io/specification) for platform-specific paths.

## Setup

1. Create a [Privy Dashboard](https://dashboard.privy.io) account
2. Create a new app to get your Requester App ID
3. Get the VeeFriends Provider App ID (contact VeeFriends team or check Privy Dashboard > Ecosystem > Integrations)
4. Set environment variables:
   ```env
   NEXT_PUBLIC_PRIVY_APP_ID=your_requester_app_id
   NEXT_PUBLIC_VF_PROVIDER_APP_ID=veefriends_provider_app_id
   ```

## Usage

### Natural Language

Just describe what you need:

- "Add VeeFriends wallet login to my Next.js app"
- "Set up cross-app authentication with VeeFriends"
- "I need to sign messages with the VeeFriends wallet"
- "Add EIP-712 typed data signing"
- "Send a transaction through the VeeFriends wallet"
- "What CSP headers do I need for Privy?"

### Reference the Skill

```
/vf-global-wallet
```

## What the Skill Covers

| Section | Description |
|---------|-------------|
| Prerequisites | Privy Dashboard setup, app ID creation |
| Environment Variables | Config validation, fail-fast pattern |
| PrivyProvider Setup | Disable embedded wallets, theme config |
| SSG Prevention | `force-dynamic` export (critical) |
| Cross-App Login | Programmatic login, error mapping |
| Wallet Address Extraction | camelCase SDK quirk, address validation |
| Sign Message | No gas, popup consent |
| Sign Typed Data (EIP-712) | BigInt usage, `as const` for TypeScript |
| Send Transaction | Gas costs, chain selection |
| Auth Guard | Route protection, redirect pattern |
| Security Headers | CSP with all Privy domains |
| Account Funding | Gas sponsorship for production apps |
| Common Gotchas | 7 documented pitfalls with solutions |
| Error Boundary | Crash recovery component |

## Reference Implementation

A complete working example is available:

- **Repo:** [vf-global-wallet](https://github.com/jeremyknows/vf-global-wallet)
- **Live:** [vf-wallet-test.vercel.app](https://vf-wallet-test.vercel.app)

## Compatibility

- Next.js 14+ (App Router)
- React 18+
- TypeScript
- `@privy-io/react-auth` v3.13.x

## Limitations

- **Users must have an existing VeeFriends account** — the SDK cannot create new accounts
- **Ethereum and Base chains only** — VeeFriends wallets are ETH-based
- **Popup-based consent** — users with aggressive popup blockers may need to allowlist your domain
- **Single concurrent operation** — only one wallet popup can be active at a time

## File Structure

```
vf-global-wallet/
├── SKILL.md          # Integration guide (loaded by AI agents)
├── LICENSE.txt       # MIT license
├── README.md         # This file (GitHub-facing documentation)
└── .gitignore        # OS artifacts
```

## License

MIT
