# crossmint-cpi-skill

A lobster.cash-compatible skill that teaches AI agents the Solana CPI inner-instruction nuance when paying x402 endpoints from Crossmint smart wallets.

## Why this exists

Crossmint smart wallets on Solana are PDAs (program-derived addresses). When an agent transfers USDC from a Crossmint wallet, the transfer is not a top-level SPL instruction — it happens inside a cross-program invocation (CPI), one level down.

This means:
- Hand-rolled SPL `TransferChecked` instructions fail with "signer not found"
- Naive x402 facilitators miss the nested transfer and reject valid payments
- No public Crossmint doc, Solana Foundation guide, or lobster.cash skill explains this

This skill teaches agents the correct pattern: use `wallet.send()` from `@crossmint/wallets-sdk`, which handles CPI wrapping, signing via the recovery signer, and gasless fee payment automatically.

## What the skill covers

- The CPI inner instruction pattern explained in plain English
- What NOT to do (the wrong-way code that fails)
- The correct `wallet.send()` recipe
- Full x402 payment loop from a Crossmint smart wallet
- Facilitator verification guidance (use `postTokenBalances`, not top-level instruction parsing)
- Decision tree for common scenarios
- Error table with causes and fixes

## Installation

```bash
npx skills add https://github.com/0xultravioleta/crossmint-cpi-skill --global --yes
```

Installs across 28+ agents: Claude Code, Cursor, Cline, Codex, OpenClaw, Gemini CLI, and more.

## Companion artifact

This skill is paired with [`crossmint-wallets-mcp`](https://github.com/0xultravioleta/crossmint-wallets-mcp) — an MCP server that exposes the same wallet primitives as tools for Claude Desktop, Continue.dev, Cline, Codex CLI, and any MCP-native client.

- The **MCP server** fills the transport gap (MCP clients can't install lobster.cash skills)
- The **CPI skill** fills the knowledge gap (no skill teaches the inner-instruction nuance)

## License

MIT — fork into `@crossmint` with zero friction.
