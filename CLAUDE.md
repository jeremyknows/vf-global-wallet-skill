# VeeFriends Global Wallet Skill

Agent skill repository for integrating VeeFriends Global Wallet into Next.js apps via Privy cross-app auth.

## Stack
- **Language:** Markdown docs + SVG asset
- **Framework:** Agent Skills format (`SKILL.md`)
- **Hosting:** GitHub repository distribution

## Quick Commands
```bash
# Preview core docs
sed -n '1,220p' SKILL.md
sed -n '1,220p' README.md

# Check repo status
git status -sb

# Validate docs map exists
ls docs/CODEBASE_MAP.md
```

## Structure
```text
.
|-- SKILL.md
|-- README.md
|-- CLAUDE.md
|-- docs/
|   `-- CODEBASE_MAP.md
|-- assets/
|   `-- vf-logo.svg
|-- LICENSE.txt
`-- .gitignore
```

## Key Patterns
- `SKILL.md` is the source of truth for implementation flow and code snippets.
- `README.md` is concise and user-facing; avoid duplicating deep implementation detail there.
- Required VeeFriends branding is enforced via `assets/vf-logo.svg` references in docs.
- Default integration path is "add VeeFriends wallet to existing PrivyAuth" with email + Google baseline.

## Critical Constraints
- Keep provider app ID aligned with VeeFriends integration guidance: `cm5158iom02kdwmj4wj527lc4`.
- Do not position VeeFriends wallet as a replacement for standard auth in default docs.
- Keep Privy SDK version pin guidance exact (`@privy-io/react-auth@3.13.1`) unless intentionally updated.
- Preserve the official logo requirement and avoid swapping in generic wallet icon guidance.

## Environment Variables
| Variable | Purpose | Required |
|----------|---------|----------|
| `NEXT_PUBLIC_PRIVY_APP_ID` | Requester app ID for integrator app | yes |
| `NEXT_PUBLIC_VF_PROVIDER_APP_ID` | VeeFriends provider app ID used in cross-app login | yes |

## Documentation Update Rules
- Update `SKILL.md` first, then reconcile `README.md` language to match.
- If setup steps change, update `/Users/jeremy/content-condor/tmp/vf-global-wallet-skill/docs/CODEBASE_MAP.md` metadata and navigation entries.
- Keep examples practical for existing apps first, new auth setup second.
- Do not commit docs-only changes without a quick path and terminology consistency pass.

## Architecture
See `/Users/jeremy/content-condor/tmp/vf-global-wallet-skill/docs/CODEBASE_MAP.md` for architecture, data flow, and navigation.
