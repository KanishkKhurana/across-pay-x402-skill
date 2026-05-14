# across-pay-x402 — Skill Package

Crosschain x402 micropayments with autodetected source-chain funding via Across Protocol.

This folder is a **skill**: a curated documentation bundle designed to be consumed by AI coding agents (Claude Code, Cursor, IDE plugins, etc.) so that any agent can integrate the `across-pay-x402` npm package into a buyer application without the human walking it through every detail.

## Audience

- **AI coding agents** that build code against the `across-pay-x402` package.
- **Engineers** who want a curated reference instead of reading `node_modules/across-pay-x402/dist/` directly.
- **Investors and stakeholders** evaluating the integration surface.

## What the package does (one-liner)

Lets a buyer call any x402-gated HTTP endpoint on a destination chain (e.g. Base) using funds it holds on **any** supported EVM origin chain. Bridging happens transparently before each paid request, only when needed.

## Folder map

```
skill/
├── SKILL.md                          # Primary entry — start here
├── README.md                         # This file
├── reference/
│   ├── api.md                        # Function signatures, return types, all exports
│   ├── configuration.md              # Every config option, default, and env var
│   ├── flow.md                       # Internal bridging flow (step-by-step)
│   ├── across-protocol.md            # The three Across HTTP endpoints used
│   └── supported-chains.md           # Chain IDs and default RPCs
├── guides/
│   ├── getting-started.md            # From `npm install` to first paid call
│   ├── wallet-options.md             # CDP vs. local private-key wallet
│   ├── polyfilling-fetch.md          # globalThis.fetch polyfill pattern
│   └── tuning-bridge-behavior.md     # gasBuffer, forceBridge, preferred chain, RPCs
├── examples/
│   ├── code-snippets.md              # Drop-in copy-paste patterns
│   └── runnable-demo.md              # Full runnable demo with env-var knobs
└── troubleshooting.md                # Common errors → fixes
```

## How to use this skill (as a human)

1. Read **`SKILL.md`** for the overview, prerequisites, and quickstart.
2. Follow **`guides/getting-started.md`** for a step-by-step walkthrough.
3. Use **`reference/*.md`** as a lookup index while writing code.
4. Use **`examples/runnable-demo.md`** as a starting template.

## How to use this skill (as an AI agent)

1. The frontmatter `description:` in `SKILL.md` is the trigger. Load `SKILL.md` first whenever the user asks about x402 payments, Across bridging, paid HTTP APIs, or 402-gated endpoints.
2. `SKILL.md` is self-sufficient for the most common integration task. Pull other files only as needed:
   - Need a function signature? → `reference/api.md`
   - Wondering what `gasBuffer` defaults to? → `reference/configuration.md`
   - Need to explain *why* the bridge picked a particular chain? → `reference/flow.md`
   - Debugging an error? → `troubleshooting.md`
3. Every file is self-contained and links to other files via relative path.

## Verifying the skill content

Every code snippet, type, default value, and environment variable in this skill is derived directly from the published package source in `node_modules/across-pay-x402/dist/`. If the package version changes, re-derive against `dist/` before trusting the docs.

Package version this skill describes: **`across-pay-x402@1.2.0`**.
