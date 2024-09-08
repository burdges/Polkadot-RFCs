# RFC-0000: Non-laziness proofs

|                 |                                                                                             |
| --------------- | ------------------------------------------------------------------------------------------- |
| **Start Date**  | Date of initial proposal                                                                    |
| **Description** | Address the lazy validator problem for bandwidth                                            |
| **Authors**     | Jeff Burdges, ...                                                                           |

## Motivation and Summary

As bandwidth is more expensive than compute, we make validators prove they have downloaded the block.  For this, we extend approval vote messages by a validator specific but publically verifiable ``non-laziness hash'' of the erasure code reconstruction.
Approval checkers double check one anothers' non-laziness hashes.  Incorrect non-laziness hashes are punished, likely slashed.

## Stakeholders

We do not require non-laziness proofs under the 2/3rds honest assumption of polkadot per se, but non-laziness proofs should remove the largest cost savings for lazy validators, under the assumption that polkadot's operating costs wind up being primarily bandwidth.

In doing so, non-laziness proofs better align incentives of nominators and validator operators, and more importantly 

## Explanation

We define `non_laziness_hash = Hs(approval_signing_public_key, Hs(unused_chunk))` where `unused_chunk` is fixed chunk from the block erasure coding, which we never send to any validator, but which we always reconstruct by virtue of our AFFT based encoder.  Also `Hs` is a truncated hash function, like `Hs(..) = black2b(..)[0..8]`.  We do include `unused_chunk` under their Merkle roots `chunks_root` for erasure code chunks.

We should avoid validator set sizes that leave us no unused chunks.  We add a new laziness punishment message which consists of a signed approval vote with an incorrect `non_laziness_hash`, the assocaited `unused_chunk`, and the copath that proves `unused_chunk` lies under `chunks_root`.  We never send `unused_chunk` over the network except in laziness punishment messages.

Approval checkers shall retain their `Hs(unused_chunk)` and check the `non_laziness_hash` of other approval checkers, sending a laziness punishment message if incorect.  We might not punish more than one lazy validator per `unused_chunk`, but we do punish one with high probability, so it likely suffices if laziness punishments exceed approval rewards.

An adversary could recompute `non_laziness_hash` more efficently than recomputing the erasure code chunks to check their Merkle root, but computing `non_laziness_hash` costs virtually nothing after recomputing the erasure code chunks, so overall this costs 16 bytes of bandwidth per approval check.

## Drawbacks, including Performance

We spend 8 bytes for every approval, so at minumum 256 bytes per core.  Assuming punishments sufficently exceed benefits, then we could shrink `Hs` below 8 bytes, even to just one single bit.  

We now require signatures for approval messages, whereas we could "almost" replace approval messages by hash preimages previously, because approval messages were empty except for context previously.  If `Hs` were only 1 bit, then a single-use Lamport signature upon that 1 bit messages requires 64 byte public keys, and 32 byte signatures.  At present, we anyways envision making this protocol impossible by using threshold randomness in tranche zero.

<!-- ## Testing, Security, and Privacy -->

<!-- ## Prior Art and References -->

## Unresolved Questions

Should laziness punishment be a simple a fee or a slash with disabling?  As always disabling maybe benefit nominators.

<!-- Provide specific questions to discuss and address before the RFC is voted on by the Fellowship. This should include, for example, alternatives to aspects of the proposed design where the appropriate trade-off to make is unclear. -->

## Alternatives

In principle, parachains could return some execution trace, but doing this properly impacts parachain execution time.  We could similarly encorporate an `export_unused_chunk` from the exports erasure coding, but this only works when blocks export data.  We think these looks much less significant, but they matter if approval voters face race conditions for their rewards.  We keep no-show timeouts long, and backpressure agressively in backing, so that approval voters never face race conditions.

We'd could do tighter economic analysis, which permits minimizing the punishment, but this seems unecessary too.  We could hide `H(unused_chunk)` somehow, like inside a Pedersen commitment, but this make the protocol much more expensive, and provides minimal benefits.

<!-- ## Future Directions and Related Material -->

