+++
title = "Solana Gossip: The Byte That Broke Everything"
date = 2026-09-30
summary = "Reverse-engineering a gossip wire protocol byte by byte, and the single missing attribute that made every packet after it garbage."
draft = true

[taxonomies]
tags = ["rust", "solana", "gossip", "networking", "serialization", "debugging"]
+++

<!--
DRAFT. Everything here comes from the material Victor gave me about sg32 and his
gossip writeup. Gaps are marked with [FILL: ...] - do not publish until they're gone.
He also has a longer mdbook version of this at docs/ in the sg32 repo; this post is
the distilled version.
-->

I wanted to run Solana without running Solana.

Not a full validator, not a delegated RPC provider — something in between. A
process that connects to the real gossip network, receives real shreds, and can
prove that a given transaction was in a given block without asking anyone to
take its word for it.

That project is [sg32](https://github.com/victorchukwuemeka/sg32). This post is
about the part nobody warns you about: the three days it took to get Agave to
send me anything at all, and why the bug was one missing attribute on one struct
field.

## Gossip in 60 seconds

Every validator runs a gossip service on UDP 8001. It has six message types:

```text
PullRequest(CrdsFilter, CrdsValue)     (0)  "Here's who I am, send me what you know"
PullResponse(Pubkey, Vec<CrdsValue>)   (1)  "Here's what I know about the network"
PushMessage(Pubkey, Vec<CrdsValue>)    (2)  "Here's some new data I just heard"
PruneMessage(Pubkey, PruneData)        (3)  "Stop sending me messages from these peers"
PingMessage(Ping)                      (4)  "Are you alive?"
PongMessage(Pong)                      (5)  "Yes, I'm alive"
```

Underneath is a CRDS table — a Conflict-free Replicated Data Type. Every validator
holds a set of `CrdsValue` entries: ContactInfo, votes, slot hashes, epoch info.
CRDS is a CRDT, so merges are order-independent and idempotent. Two validators can
merge the same entries in any order and converge, no coordination required.

That's the whole idea. Information propagates by gossip, and the network is
self-healing because merging is commutative.

The conversation looks like this:

1. You send `Ping` → validator replies `Pong`. Both are 132 bytes. You know UDP works.
2. You send `PullRequest(your ContactInfo)` → the entrypoint queues it.
3. The entrypoint sends *you* a `Ping` → you reply `Pong`. Now it knows you're responsive.
4. The entrypoint sends `PullResponse` → you learn about other validators.
5. You repeat step 2 every ~5 seconds to stay in the set.

Steps 1 and 3 are liveness checks. Step 2 is how you get admitted. Step 4 is the
payload.

## Where it broke

Step 1 worked immediately. `Ping` and `Pong` are trivially simple — a magic number
and a bytemuck-encoded struct. I had 132 bytes going both ways within the first
evening.

Step 2 is where it died. I sent a `PullRequest`. Nothing came back. Not an error —
*nothing*. The entrypoint received the datagram, and silently ignored it.

Silence is the worst possible failure mode. A rejected connection tells you
something. A dropped packet that never gets an error just tells you your model of
the protocol is wrong, without saying where.

## Phase 1: assuming the request was malformed

My first assumption was that the entrypoint received my packet, couldn't parse
my `ContactInfo`, and dropped it. The fix I reached for was making my struct
match Agave's exactly — field for field, in order.

This is where it gets tedious. Agave's `ContactInfo` is not small. It carries
socket info for a dozen different subsystems — tpu, tvu, repair, pubsub, vote,
shred_fetch, forwarder — plus the pubkey, the FQDN, and an `unknown` field
literally typed as `Vec<u8>` to absorb future additions without a breaking change.

My struct was over 300 bytes. Agave's is 65.

That gap is almost entirely socket structs, and here's the trap: those are
`UdpSocket` wrappers that serialize into **7 bytes**, not a port number. A
`SocketAddr` is a family tag plus a 4-byte IP plus a 2-byte port. If you model
that as "a port, I'll fill in the IP later," you are wrong by enough bytes to
corrupt everything after it.

## Phase 2: the Version struct

I checked the envelope rather than the payload. Every gossip message is wrapped,
and the first thing in the stream is a `Protocol` enum discriminant — a u32 saying
which of the six message types this is.

I was sending the wrong size. The `Version` struct in the envelope serialized to
17 bytes on my side and 12 on Agave's. Five bytes of pure nothing, injected at the
front of the message, shifting everything downstream by five.

I fixed the struct.

Nothing changed.

## Phase 3: the CrdsValue hash

`CrdsValue` is a tagged union — ContactInfo, Vote, SlotHashes, EpochData — and
before the union there's a 32-byte hash. That hash is the *label*: it's derived
from the serialized contents of the value, and it's what CRDS uses to deduplicate
and to detect conflicts.

I had the hash as 32 bytes of poison. Not wrong data — garbage that happened to be
the right length. My hash didn't match the hash of the value it was supposed to
label, so every entry I pushed was internally inconsistent.

I fixed the hash.

Nothing changed.

## Phase 4: the actual root cause

Here's the bug, and it's the whole reason this post exists.

`ContactInfo` has a field called `wallclock` — a `u64` holding a UNIX timestamp.
In Agave, that field is **not** serialized as a plain integer. It carries a
`serde_varint` attribute:

```rust
#[serde(with = "serde_varint")]
pub wallclock: u64,
```

Bincode's default for a `u64` is a fixed 8 bytes, little-endian, every time.
`serde_varint` replaces that with a variable-length encoding — 7 bits per byte,
with the high bit set on every byte except the last. A small number is one byte.
A large one is up to eight.

My `ContactInfo` was missing that attribute. So I was writing 8 fixed bytes where
Agave expected a varint.

And here's the part that cost me the time. Agave's varint reader looked at the
**first** byte of my 8-byte field. If that byte's high bit was clear, it concluded
"one-byte varint, value complete." It read one byte. It moved on.

The other seven bytes — the rest of my timestamp — were now sitting in the stream
where Agave expected the *next field*. So Agave read my wallclock as a
one-byte value, then read the remaining seven bytes of my timestamp as the
beginning of the next struct, and from that point forward it was reading a message
that was no longer aligned to any field boundary.

One byte of misalignment at the front of a serialized struct doesn't corrupt one
field. It corrupts *every field after it*, and it does it silently, because
deserialization of a byte stream has no idea it's lost sync. You don't get an
error. You get a plausible-looking `ContactInfo` full of nonsense, and then a
`PullRequest` that gets dropped because the pubkey doesn't correspond to anything
real.

**The lesson, and the reason I spent a day on the envelope before looking at the
payload:** if the bytes on the wire are wrong, the struct definitions you build
on top of them are also wrong, and no amount of fixing the struct definitions will
help. I had three plausible bugs, fixed all three correctly, and observed nothing
change — three times — because the actual bug was a serialization attribute that
none of my diffs touched.

Verify the wire format before you trust the types. Dump bytes. Count them.

## The conversation, byte by byte

Once the attribute was there, this is what the devnet entrypoint and sg32
actually exchanged:

```text
Message 1  Ping             132 bytes    sg32 -> devnet
Message 2  Pong             132 bytes    devnet -> sg32
Message 3  PullRequest     1232 bytes    sg32 -> devnet
Message 4  Ping             132 bytes    devnet -> sg32
Message 5  PullResponse    505-1232      devnet -> sg32
```

132 bytes is the floor — the envelope plus a `Ping` is a fixed-size message, which
is why liveness checks are cheap to answer and why Agave can afford to Ping every
new peer it admits.

`PullRequest` at 1232 bytes is interesting. That's a padded message, and the
padding is deliberate: these travel over UDP, and a single IP datagram has to fit
inside a conservative MTU so it doesn't get fragmented. Fragmented UDP is
unreliable in exactly the way gossip can't tolerate. So the wire format is
constrained by the transport underneath it, and the serializer has to work
backwards from an MTU budget.

`PullResponse` varies — 505 to 1232 bytes — because it's carrying a variable
number of CRDS entries, and it's capped at the same MTU limit.

[FILL: paste an actual hexdump of a PullResponse here, from the logs in the sg32
repo. Annotated, showing where the discriminant is and where your ContactInfo
starts. The repo has logs/ committed.]

## What I still get wrong

[FILL: your notes on this — you have a section on this in the mdbook. This is the
part that makes the post credible, don't skip it.]

Some things I know are incomplete: [FILL — pruning, the repair protocol, whatever
your mdbook says you haven't solved yet.]

## Run it yourself

```bash
git clone https://github.com/victorchukwuemeka/sg32
cd sg32
cargo run --release
```

Defaults point at devnet. Point it at mainnet with:

```bash
cargo run --release -- --entrypoint entrypoint.mainnet-beta.solana.com:8001
```

The full byte-level walkthrough, the file-by-file change log, and the complete
devnet conversation are in the repo's `docs/` mdbook.

## Related

- [How Solana Gossip Really Works: A Byte-Level Journey Into the Devnet](https://github.com/victorchukwuemeka/sg32)
  — the long version, with every reference file in the Agave source
