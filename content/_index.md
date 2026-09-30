+++
title = "Victor Chukwuemeka"
description = "Low-level systems engineer. I build networked protocols in Rust from the bytes up — gossip, wire formats, erasure coding, and light clients."
+++

I build low-level systems in Rust. Networking, wire formats, serialization,
erasure coding — the layer where a protocol stops being a spec and starts being
bytes on a socket.

My main project is [sg32](https://github.com/victorchukwuemeka/sg32), a Solana
light node. It's a single binary that binds four sockets, speaks a gossip wire
protocol to ~thousands of live peers, receives network shreds, reconstructs whole
blocks with Reed-Solomon erasure coding, keeps a CRDT table in memory, writes
blocks to disk, and serves JSON-RPC with cryptographic proofs of transaction
inclusion. No full node. No RPC provider in the trust path.

I got here by reverse-engineering Agave's gossip serialization byte by byte until
the devnet started answering, and I have merged changes into SPL Token-2022,
SPL Token, Pinocchio and Agave.

Cross-chain bridges are what I use this for. They're not what I do.
