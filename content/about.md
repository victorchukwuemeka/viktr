+++
title = "About"
description = "Low-level systems engineer. Networking, wire formats, and distributed systems in Rust."
+++

## What I do

I build low-level networked systems in Rust. The interesting problems are at the
bottom: how bytes get laid out on the wire, what a protocol does when a packet
arrives short, how you recover data from partial data, and what a system assumes
when nobody's watching.

Concretely, that means:

- **Wire protocols and serialization** — bincode, `serde_varint`, `short_vec`,
  and what happens to a byte stream when a field is one byte wider than the reader
  expects
- **Networking** — UDP, MTU budgets, gossip, CRDT replication, port allocation,
  the failure modes of a transport with no delivery guarantee
- **Erasure coding** — Reed-Solomon forward error correction, reconstructing
  blocks from partial shred sets
- **Light clients** — proving things about a blockchain without running a
  validator and without trusting an RPC provider
- **Serialization debugging** — dumping bytes and counting them until the model
  matches reality

## The main project

[sg32](https://github.com/victorchukwuemeka/sg32) is a Solana light node. It grew
out of an attempt to speak the gossip protocol to devnet by hand, and it ended up
as a standalone networked system:

```text
┌─────────────────────────────────────────────────────┐
│                   sg32 (single binary)               │
├─────────────────────────────────────────────────────┤
│                                                      │
│  Gossip ──> Repair ──> FEC Recovery ──> Deshredder  │
│  (:8001)    (:8003)     (Reed-Solomon)   Entries→TXs │
│                                              │       │
│                                              ▼       │
│  RPC ←── Merkle Tree ←── Deshredder                 │
│  (:8899)                                            │
│    │                                                 │
│    ▼                                                 │
│  Memory (ring buffer) + Flat File (disk)            │
│                                                      │
└─────────────────────────────────────────────────────┘
```

It speaks the gossip protocol to the live devnet, maintains a CRDS table of
validator ContactInfo, receives real shreds over TVU, reconstructs blocks with
Reed-Solomon erasure coding, and serves trustless Merkle proofs of transaction
inclusion over JSON-RPC. Recovered 21 of 32 data shreds in live testing — below
what the erasure code needs, and it still reconstructed the block.

Building it meant reverse-engineering Agave's wire format from its source, byte by
byte, and being wrong three times in instructive ways. [That story is here.](https://victorchukwuemeka.github.io/sg32/)

## Where it's applied

At Odinala I build cross-chain bridge infrastructure. The live V1 locks SOL on
Solana and mints wrapped SOL on EVM destinations through a relayer. V2 removes
the trusted relayer: sg32 produces a Merkle proof of inclusion, SP1 turns that into
a zkVM proof, and a verifier contract on the EVM side checks it before minting.
Nobody has to take a relayer's word for anything.

The bridge is the application. sg32 is the work.

## Open source

Merged changes into [SPL Token-2022](https://github.com/solana-program/token-2022/pull/1117)
and [SPL Token](https://github.com/solana-program/token/pull/118), and
contributed to Pinocchio and Agave.

## How I got here

B.Sc. in Biotechnology from FUNAI, 2022. Software came after, self-directed: Rust,
then Solana, then contributing to the ecosystem's own repositories because reading
them wasn't enough to understand them.

The Solana Turbine 3 cohort in 2024 was the turning point — validator internals,
the SVM, and a room full of people who already knew what they were talking about.

## Elsewhere

- [GitHub](https://github.com/victorchukwuemeka)
- [X](https://twitter.com/viktr_sol)
- [Email](mailto:chukwuemekavictor693@gmail.com)
