---
name: crossmint-cpi
version: 0.1.0
description: Teaches AI agents the Solana CPI inner-instruction pattern required to pay x402 endpoints (or transfer tokens) from a Crossmint smart wallet — including what NOT to do, and why the high-level `wallet.send()` SDK method is the right primitive.
triggers:
  - crossmint
  - x402
  - solana
  - smart wallet
  - cpi
  - inner instruction
  - pay endpoint
  - wallet.send
  - TransferChecked
compatible_clients:
  - lobster-cash
  - claude-code
  - cursor
  - openclaw
---

# crossmint-cpi

> How to pay x402 endpoints (or send any SPL token) from a Crossmint smart
> wallet on Solana — and why the "obvious" hand-rolled transaction fails.

## When to invoke this skill

Invoke this skill when the user asks an AI agent to:

- Pay an x402-protected endpoint using a Crossmint smart wallet on Solana
- Transfer USDC (or any SPL token) out of a Crossmint wallet
- Debug "signer not found", "missing signature", or "invalid account owner" errors when constructing Solana transactions against a Crossmint wallet address
- Understand why a naive x402 facilitator doesn't see a Crossmint wallet's
  payment even though the on-chain balance changed

## Core insight (plain English)

A Crossmint smart wallet on Solana is not a regular user account. It's a
**program-derived address** (PDA) owned by the Crossmint wallet program.
Its USDC is held in a token account whose authority is the wallet PDA
itself — not a keypair you can sign with.

Think of it like a **corporate treasury account**: the business owns the
money, but a single employee can't just walk up and write a check from
it. Transfers have to go through the corporation's own approval process.
The money moves — but the signature on the receipt is the corporation's,
not the employee's.

For a Crossmint wallet, the "corporation" is the Crossmint wallet program,
and its "approval process" is a **cross-program invocation (CPI)**:

1. You send a transaction that calls the Crossmint wallet program with an
   "execute transfer" instruction.
2. The Crossmint program, using its program authority derived from the
   wallet PDA seeds, CPIs into the SPL token program with a `TransferChecked`
   instruction.
3. The SPL token program sees the valid PDA signer and moves the funds.

The SPL transfer happens as an **inner instruction** nested inside the
Crossmint program invocation — not as a top-level instruction of your
transaction.

This is the single most important fact about Crossmint smart wallet
payments on Solana. If you skip it, everything else looks broken.

## What NOT to do

The first thing most developers try when they see "move USDC out of a
Crossmint wallet" is to hand-build a regular SPL transfer:

```typescript
//  WRONG — will fail with "signer not found" or "missing signature"
import { createTransferCheckedInstruction } from "@solana/spl-token";

const instruction = createTransferCheckedInstruction(
  sourceAta,   // the Crossmint wallet's USDC ATA
  usdcMint,
  destAta,
  walletPubkey, //  the wallet PDA — CANNOT SIGN DIRECTLY
  amount,
  6,
);

const tx = new Transaction().add(instruction);
// ... sign with what? The PDA has no private key.
```

The problem: `createTransferCheckedInstruction` expects the `authority`
parameter to be something that can produce a signature. The Crossmint
wallet PDA can't — its authority comes from the wallet program, not a
keypair. Any attempt to sign with the API key or recovery secret directly
fails because the network doesn't know about Crossmint's off-chain signers.

## What TO do

Use `Wallet.send()` from `@crossmint/wallets-sdk`. It handles the CPI
wrapping internally:

```typescript
//  CORRECT — let the SDK wrap the transfer in a CPI call
import { createCrossmint, CrossmintWallets } from "@crossmint/wallets-sdk";

const crossmint = createCrossmint({ apiKey: process.env.CROSSMINT_API_KEY! });
const wallets = CrossmintWallets.from(crossmint);

// Load the wallet — IMPORTANT: pass recovery in the args so the returned
// Wallet has its internal recovery field populated. The TypeScript
// WalletArgsFor type omits recovery, but the runtime reads it.
const wallet = await wallets.getWallet(walletAddress, {
  chain: "solana",
  recovery: { type: "server", secret: process.env.CROSSMINT_RECOVERY_SECRET! },
} as any);

// Send — the SDK builds the transaction, signs via the recovery signer,
// submits, and waits for confirmation. Amounts are in decimal human units.
const tx = await wallet.send(recipientAddress, "usdc", "0.01");

console.log(tx.hash);         // Solana signature
console.log(tx.explorerLink); // https://explorer.solana.com/tx/...
```

Under the hood, `wallet.send()` constructs a transaction whose top-level
instruction invokes the Crossmint wallet program. The Crossmint program,
running on-chain, CPIs into the SPL token program with the correct
authority. The resulting transaction is gasless to you (Crossmint's
relayer pays the network fee).

## Paying an x402 endpoint

This is the same flow with three extra steps bolted on at the protocol
level:

```typescript
// 1. Fetch the URL and expect a 402 Payment Required response
const probe = await fetch("https://your-x402-endpoint.example/data");
if (probe.status !== 402) throw new Error("Expected 402, got " + probe.status);

// 2. Parse the PaymentRequired body (x402 2.x spec)
const paymentRequired = await probe.json() as {
  x402Version: number;
  accepts: Array<{
    scheme: string;
    network: string;       // e.g. "solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp"
    asset: string;          // USDC mint address
    amount: string;         // atomic units (1,000,000 = 1 USDC)
    payTo: string;          // recipient Solana address
  }>;
};

// 3. Pick the first accepts[] entry on a network you support
const requirement = paymentRequired.accepts.find((r) =>
  r.network.toLowerCase().startsWith("solana"),
);
if (!requirement) throw new Error("No Solana requirement in 402 body");

// 4. Convert atomic amount → decimal and pay via wallet.send
const decimalAmount = (Number(requirement.amount) / 1_000_000).toString();
const tx = await wallet.send(requirement.payTo, "usdc", decimalAmount);

// 5. Build the X-PAYMENT header with the transaction signature and
//    replay the request
const paymentHeader = Buffer.from(
  JSON.stringify({
    x402Version: paymentRequired.x402Version,
    accepted: requirement,
    payload: { transactionSignature: tx.hash },
  }),
).toString("base64");

const paid = await fetch("https://your-x402-endpoint.example/data", {
  headers: { "X-PAYMENT": paymentHeader },
});

console.log(await paid.json());
```

That's the full x402 payment loop. The heavy lifting — CPI wrapping,
signing, gas payment, confirmation — is all inside `wallet.send()`. Your
code only needs to handle the protocol glue.

## What this means for x402 facilitators

If you are BUILDING an x402 facilitator that needs to verify Crossmint
smart-wallet payments, you cannot just read `transaction.message.instructions`
on the confirmed tx and look for SPL transfers. You will miss every
single Crossmint payment, because the actual SPL transfer lives one level
deeper — as an **inner instruction** of the Crossmint wallet program
invocation.

Instead, verify by reading `postTokenBalances` and `preTokenBalances`
from the transaction meta, and check that the destination ATA balance
increased by at least the required amount. This is what
[Corbits](https://www.corbits.dev) does, and it is the approach
the companion
[crossmint-wallets-mcp](https://github.com/0xultravioleta/crossmint-wallets-mcp)
paywall server uses in its verification path. Example:

```typescript
const tx = await connection.getTransaction(signature, {
  commitment: "confirmed",
  maxSupportedTransactionVersion: 0,
});

const post = tx?.meta?.postTokenBalances ?? [];
const pre = tx?.meta?.preTokenBalances ?? [];

const postBal = post.find(
  (b) => b.mint === USDC_MINT && accountKeys[b.accountIndex] === merchantAta,
);
const preBal = pre.find((b) => b.accountIndex === postBal?.accountIndex);

const delta = BigInt(postBal?.uiTokenAmount.amount ?? "0")
            - BigInt(preBal?.uiTokenAmount.amount ?? "0");

if (delta < requiredAmount) throw new Error("insufficient payment");
```

Naive facilitators that only parse top-level instructions will see an
opaque Crossmint program invocation and assume nothing happened.

## Quick decision tree for agents

```
Need to move tokens out of a Crossmint Solana wallet?
├── Standard token transfer (USDC, SOL, etc.)?
│   └── Use wallet.send(recipient, tokenSymbol, decimalAmount)
├── Pay an x402 endpoint?
│   └── Parse the 402 body, then use wallet.send() + X-PAYMENT header
├── Complex multi-instruction Solana transaction?
│   └── Use SolanaWallet.sendTransaction({ serializedTransaction })
│       Build the transaction so its instructions INVOKE the Crossmint
│       wallet program. Do NOT try to sign SPL instructions directly
│       against the wallet PDA.
└── Interact with a program the Crossmint wallet hasn't wrapped yet?
    └── Open an issue on the Crossmint SDK repo. Do not attempt to
        bypass the wallet program.
```

## Common errors and what they actually mean

| Error message                                 | Real cause                                                                 |
|------------------------------------------------|----------------------------------------------------------------------------|
| `Missing signature for public key [PDA]`       | You tried to sign an SPL instruction with the wallet PDA as authority. Use `wallet.send()` instead. |
| `No signer is set. Server wallets require wallet.useSigner()` | You loaded the wallet via `getWallet(locator, args)` without passing `recovery` in the args. See the workaround in the "What TO do" section above. |
| `Cannot read properties of undefined (reading 'startsWith')` | Same root cause as above — the `#recovery` field was never populated on the wallet instance because `WalletArgsFor` omits `recovery` from the type. Pass it via a cast. |
| `Token account does not exist`                 | The source or destination ATA hasn't been created yet. `wallet.send()` should handle source-side creation; for destination, the recipient's wallet needs a USDC ATA. |
| Facilitator says "no payment detected" but the transaction succeeded on-chain | The facilitator is parsing top-level instructions only. It needs to inspect `postTokenBalances` for inner CPI transfers. |

## Companion artifact

This skill pairs with
[`crossmint-wallets-mcp`](https://github.com/0xultravioleta/crossmint-wallets-mcp),
an MCP server that exposes `crossmint_create_wallet`, `crossmint_get_balance`,
`crossmint_transfer_token`, and `crossmint_pay_x402_endpoint` as tools for
any MCP-native client (Claude Desktop, Continue.dev, Cline, Codex CLI).

The MCP server implements the `wallet.send()` pattern described here in
`src/core/pay-x402-endpoint.ts` and `src/core/transfer-token.ts`. The
paywall server in `demo/paywall-server.ts` is an example of the
`postTokenBalances`-based facilitator verification pattern. Read both for
a complete working reference.

## References

- **Crossmint wallets SDK**: https://www.npmjs.com/package/@crossmint/wallets-sdk
- **x402 protocol**: https://x402.org
- **@x402/core types**: https://www.npmjs.com/package/@x402/core
- **@x402/svm (Solana)**: https://www.npmjs.com/package/@x402/svm
- **Companion MCP server**: https://github.com/0xultravioleta/crossmint-wallets-mcp
- **Solana CPI docs**: https://solana.com/docs/core/cpi
- **MIT License** — so Crossmint can fork this skill into the official
  lobster.cash skill directory without friction.
