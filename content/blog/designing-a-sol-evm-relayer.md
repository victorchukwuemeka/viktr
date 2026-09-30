+++
title = "Designing a Relayer for SOL to EVM Bridging"
date = 2026-09-30
summary = "Notes on the relayer architecture behind a cross-chain bridge: when to move, how to stop double-spending, and what happens when a relayer VM dies mid-transfer."
draft = true

[taxonomies]
tags = ["rust", "solidity", "infrastructure", "bridges"]
+++

<!--
DRAFT — this is a skeleton built from the project description on your portfolio.
Everything in [brackets] needs your real numbers and your real decisions.
Nothing here is invented fact; delete what doesn't apply to how you actually built it.
-->

Bridges get talked about as if the hard part is moving the token. It isn't. Moving
the token is the easy part — you call a contract, you transfer, you're done. The
hard part is everything around it: knowing when you're allowed to move it, proving
you didn't move it twice, and surviving a relayer dying halfway through.

This is what I learned building a bridge that moves SOL to Base, Polygon, and
Ethereum mainnet.

## The two halves of a bridge

A bridge is two contracts and a process:

1. **Lock or burn on the source chain** — the user's SOL leaves Solana and enters
   an escrow or gets burned.
2. **Mint or release on the destination chain** — the equivalent amount appears on
   the EVM side.

The first step is a user transaction. The second step is *our* problem. Nobody is
going to manually mint tokens for people. That's the relayer.

## Why a relayer at all

I considered the obvious alternative: let users submit the proof themselves. They'd
pay gas on the destination chain, and the contract would verify the lock event and
mint.

I went with a relayer instead. Three reasons:

- **UX.** A bridge where users have to submit a second transaction on a different
  chain, waiting for finality, is a bridge most people abandon halfway.
- **Cost.** Users pay once, on the source chain. The relayer operator eats the
  destination gas.
- **Failure recovery.** When something goes wrong, we can fix it centrally instead
  of hoping users come back to claim.

The trade is real though: a relayer is a trusted component, and that's the part
worth thinking about hardest.

## Replay protection

The first thing you get wrong is double-minting. If a user can present the same
deposit proof twice, they mint twice. Every bridge contract needs the same three
defenses:

**A nonce per deposit.** The source-chain transaction hash is unique. Minting
consumes it. A second attempt for the same hash reverts.

**A domain separator.** If the same contract is deployed at the same address on
multiple chains — which happens constantly with CREATE2 — a signature valid on one
chain is valid on all of them. EIP-712 domain separation binds the signature to a
specific chain ID and contract address:

```solidity
bytes32 constant DOMAIN_SEPARATOR =
    keccak256(
        abi.encode(
            keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
            keccak256(bytes("CrossBridge")),
            bytes1(0x01),
            CHAIN_ID,
            address(this)
        )
    );
```

**A deadline.** Signature-based minting without an expiry is a permanent liability.
Any leak of a signed message becomes an open mint forever. This was the single
biggest finding in the audit of our EVM contracts.

```solidity
require(block.timestamp <= deadline, "signature expired");
```

## What happens when a relayer dies

This is the part nobody writes about.

Our relayer runs across multiple VMs — I don't want to claim we built a
nondurable system, because we didn't — and the failure mode is straightforward:
the relayer watches the source chain for deposit events, and if a VM dies between
"saw the event" and "minted on destination," the user's SOL is locked with nothing
to show for it.

Three mitigations, in the order we needed them:

**1. Event-driven scanning, not polling intervals.** Scanning a cursor instead of
re-scanning a window. Re-scanning a window is how you get duplicate processing and
missed confirmations at the same time.

**2. Idempotent state, not idempotent code.** The mint function is safe to call
twice for the same deposit — the second call reverts on the consumed nonce, but
it doesn't corrupt anything. The relayer treats a revert as "already handled" and
moves on, rather than retrying into a stuck loop.

**3. A reconciliation pass.** [This is the part I still think is the weakest part
of the system] — a scheduled job that compares source-chain deposits against
destination-chain mints and flags the difference. Not automatic recovery, just
detection. [Fill in how you actually handled unresolved deposits — this is the
honest part of the post and the reason anyone would read it.]

## Confirmation thresholds

SOL finality and EVM finality are different, and picking a threshold is a real
tradeoff rather than a constant you can look up.

[Fill in your actual numbers here and, more importantly, why.] Waiting for full
EVM finality before minting is safe and slow. Waiting a fixed block count is fast
and occasionally wrong, and the failure is not a failed transaction — it's
minted tokens backed by a deposit that got reorganized out of existence.

## What I'd do differently

[Your call. Mine would probably be: build the reconciliation tooling first instead
of last, because the relayer is going to have downtime and the interesting question
isn't whether, it's what happens when it does.]

## Where the code is

[Link the repo, or a cleaned-up version of it. A bridge post with no code in it
is a blog post.]
