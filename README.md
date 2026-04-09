# crossmint-cpi-skill

> A lobster.cash-compatible skill that teaches AI agents the Solana CPI
> inner-instruction nuance when paying x402 endpoints from Crossmint smart
> wallets.

**Status:** scaffold — full content populated during Phase 2G.

## Why this skill exists

Crossmint smart wallets on Solana are PDAs (program-derived addresses) owned
by the Crossmint wallet program. When an AI agent constructs a Solana
transaction that transfers USDC out of a Crossmint wallet to pay an x402
endpoint, the transfer cannot be a direct SPL token transfer — it must be
issued as a cross-program invocation (CPI) *inner instruction* from the
Crossmint wallet program, signed by the Crossmint API on the agent's behalf.

None of the 13 currently certified lobster.cash skills cover this technical
nuance. Agents that don't know about it will construct malformed transactions
and waste cycles debugging "signer not found" errors.

## What this skill teaches

1. **The difference between an SPL token transfer and a CPI inner
   instruction** — when and why each is required.
2. **How to construct the correct transaction** for a Crossmint
   wallet-to-EOA USDC transfer that clears an x402 payment challenge.
3. **How to hand the unsigned transaction to `@crossmint/wallets-sdk`**
   via `SolanaWallet.sendTransaction({ serializedTransaction })` so the
   Crossmint API signs it server-side with the recovery signer.
4. **How to interpret the transaction signature** returned by Crossmint
   and replay the x402 request with an `X-PAYMENT` header.

## Companion artifact

This skill is paired with
[`crossmint-wallets-mcp`](https://github.com/0xultravioleta/crossmint-wallets-mcp),
an MCP server that exposes the same primitives as tools for Claude Desktop,
Continue.dev, Cline, Codex CLI, and any MCP-native client. Together they
cover the two gaps in Crossmint's agentic commerce stack — the MCP server
fills the transport gap for MCP-native clients, and this skill fills the
knowledge gap for any LLM that needs to hand-roll the transaction.

## Installation

Populated during Phase 2G per
`_internal/planning/plans/03-mcp-build-and-skill-plan.md`.

## License

MIT — so that Crossmint can fork this repo into the official
`@crossmint` organization with zero friction.
