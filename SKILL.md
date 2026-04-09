---
name: crossmint-cpi
version: 0.1.0
description: Teaches AI agents the Solana CPI inner-instruction pattern required to pay x402 endpoints from a Crossmint smart wallet.
triggers:
  - crossmint
  - x402
  - solana
  - smart wallet
  - cpi
  - inner instruction
compatible_clients:
  - lobster-cash
  - claude-code
  - cursor
  - openclaw
---

# crossmint-cpi — Skill (scaffold)

> **Status:** scaffold — full content populated during Phase 2G.

## When to invoke this skill

Invoke when the user asks an agent to:

- Pay an x402-protected endpoint from a Crossmint smart wallet on Solana
- Transfer USDC out of a Crossmint wallet (any destination)
- Debug "signer not found" / "missing account" errors when constructing
  Solana transactions against a Crossmint PDA

## Core insight

*(Populated in Phase 2G.)*

## Transaction recipe

*(Populated in Phase 2G.)*

## Working example

*(Populated in Phase 2G — will link to the demo in `crossmint-wallets-mcp`.)*

## References

- Crossmint wallets SDK: https://www.npmjs.com/package/@crossmint/wallets-sdk
- x402 protocol: https://x402.org
- Companion MCP server: https://github.com/0xultravioleta/crossmint-wallets-mcp
