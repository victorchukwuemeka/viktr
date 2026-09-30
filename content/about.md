+++
title = "About"
description = "Protocol engineer working on Solana internals in Rust."
+++

## What I do

I work on Solana protocol internals in Rust. I have merged changes into
[SPL Token-2022](https://github.com/solana-program/token-2022/pull/1117),
[SPL Token](https://github.com/solana-program/token/pull/118),
[Pinocchio](https://github.com/anza-xyz/pinocchio) and Agave.

My main project is [sg32](https://github.com/victorchukwuemeka/sg32) — a Solana
light node. It connects to the devnet gossip network, receives live shreds over
TVU, reconstructs blocks using Reed-Solomon erasure coding, and serves
trustless Merkle proofs of transaction inclusion over JSON-RPC. One binary, one
port, no full node and no RPC provider in the trust path.

At Odinala I build cross-chain bridge infrastructure — a live SOL-to-EVM bridge,
and a trustless V2 where a Merkle proof feeds an SP1 zkVM proof that an EVM
verifier contract checks before minting.

## How I got here

I have a B.Sc. in Biotechnology from FUNAI, graduated 2022. Software came after,
self-directed: Rust, then Solana, then contributing to the ecosystem's own
repositories because reading them wasn't enough to understand them.

The Solana Turbine 3 cohort in 2024 was the turning point — validator internals,
the SVM, and a team of people who actually knew what they were talking about.

## What I write about

Byte-level debugging, wire formats, the gap between what a protocol's docs say
and what its implementation actually does, and the design decisions behind
light clients and bridges.

These are notes from building things, not tutorials. If something I got wrong is
useful to you, that's the part I care about writing down.

## Elsewhere

- [GitHub](https://github.com/victorchukwuemeka)
- [X](https://twitter.com/viktr_sol)
- [Email](mailto:chukwuemekavictor693@gmail.com)
