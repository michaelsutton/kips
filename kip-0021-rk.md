```
KIP: 21-RK
Title: KIP-21 Reference Design (rusty-kaspa)
Status: Companion to KIP-21
Created: 2026-05-28
```

## Purpose

This document pins the byte-level encodings, SMT representation, storage layout, cache model, IBD wire shape, and witness-store layout for the rusty-kaspa reference implementation of [KIP-21](./kip-0021.md). The reference implementation lives at [https://github.com/kaspanet/rusty-kaspa](https://github.com/kaspanet/rusty-kaspa).

This document describes the **final, post-activation** design as it ships in a single hard fork on mainnet. Pre-activation legacy behavior is out of scope here.

KIP-21 specifies *what* is committed and *why*; this document specifies *how* the bytes are arranged and *how* the implementation stays correct under reorgs, IBD, and pruning so independent implementations produce the same hashes and the same on-disk shape.

All hash functions are BLAKE3 in keyed mode: `H_tag(m) = BLAKE3(key = tag_padded_to_32, input = m)`, where `tag_padded_to_32` is the ASCII bytes of the tag zero-padded on the right to 32 bytes.

## 1. Hash Domain Table (Normative)

| Symbol in KIP-21 | Domain tag (ASCII) | Used for |
| --- | --- | --- |
| `H_seq` | `SeqCommitmentMerkleBranchHash` | Generic two-input branch hash. Used for `SeqCommit`, `SeqStateRoot`, `payload_and_ctx_digest`, activity-digest Merkle branches, miner-payload Merkle branches. |
| `H_lane_key` | `SeqCommitLaneKey` | `lane_key(lane_id)` ([§2](#2-lane-key)). |
| `H_lane_tip` | `SeqCommitLaneTip` | `TipUpdateHash` ([§5](#5-tip-update-hash)). |
| `H_activity_leaf` | `SeqCommitActivityLeaf` | Per-tx activity leaf ([§3](#3-activity-leaf-and-digest)). |
| `H_mergeset_context` | `SeqCommitMergesetContext` | `ctx_hash(B)` ([§4](#4-mergeset-context-hash)). |
| `H_activity_root` | `SeqCommitActivityRoot` | Wrap around `ActiveLanesRoot` ([§7](#7-activity-root)). |
| `H_leaf` | `SeqCommitActiveLeaf` | SMT leaf hash ([§2.2](#22-smt-leaf-payload-normative)). |
| `H_node` | `SeqCommitActiveNode` | SMT internal node hash ([§2.1](#21-slo-single-leaf-optimization-normative)). |
| `H_collapsed_node` | `SeqCommitActiveCollapsedNode` | SLO collapsed-node hash ([§2.1](#21-slo-single-leaf-optimization-normative)). |
| `H_miner_payload` | `PayloadDigest` | Mergeset miner payload bytes ([§6](#6-mergeset-miner-payload-commitment)). |
| `H_miner_payload_leaf` | `SeqCommitMinerPayloadLeaf` | Per-mergeset-block miner-payload leaf ([§6](#6-mergeset-miner-payload-commitment)). |

`H_seq(x, y)` is shorthand for `H_seq(x || y)` where `x` and `y` are 32-byte values.

Reference: [`crypto/hashes/src/hashers.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/crypto/hashes/src/hashers.rs), [`consensus/seq-commit/src/hashing.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/seq-commit/src/hashing.rs).

## 2. Lane Key

```
lane_key(lane_id) = H_lane_key(lane_id)        // 32-byte SMT key
```

`lane_id` is the canonical 20-byte `tx.subnetwork_id` ([KIP-21 §2.1](./kip-0021.md#21-canonical-lane-id-encoding-normative)). The result is interpreted MSB-first as a path into a depth-256 SMT.

All mergeset coinbase transactions share the dedicated coinbase subnetwork ID, so they share a single lane key. Only the selected-parent coinbase enters `AcceptedTxList(B)` and contributes to the coinbase lane's activity digest; non-selected-parent coinbases are committed separately via the mergeset miner payload commitment ([§6](#6-mergeset-miner-payload-commitment)).

The coinbase lane key is precomputed as a constant: see `COINBASE_LANE_KEY` in [`consensus/seq-commit/src/hashing.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/seq-commit/src/hashing.rs).

### 2.1 SLO (Single-Leaf-Optimization) (Normative)

A subtree containing exactly one leaf is hashed and stored as a single *collapsed* node, not as a full chain of 256 internal nodes. The collapsed hash uses a separate domain `H_collapsed_node`:

```
collapsed_node_hash(lane_key, leaf_hash) = H_collapsed_node( lane_key || leaf_hash )
```

`H_collapsed_node` is a domain disjoint from `H_node` and `H_leaf` so collapsed, branch, and leaf preimages can never be confused (second-preimage separation).

Branching uses `H_node(left_child || right_child)`. Empty subtrees use the canonical chain:

```
EMPTY_0     = ZERO_HASH
EMPTY_{i+1} = H_node( EMPTY_i || EMPTY_i )
```

Empty-subtree hashes `EMPTY_0..EMPTY_256` are precomputed at build time; the empty-tree root is `EMPTY_256`. See [`crypto/smt/build.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/crypto/smt/build.rs).

SMT proofs follow [`crypto/smt/src/proof.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/crypto/smt/src/proof.rs): a 32-byte sibling-empty bitmap compresses the proof, and a `ProofTerminal` discriminator distinguishes proofs that stop at the leaf level (`Full`) from those that stop at a collapsed subtree containing the queried key (`Collapsed`) or a foreign key (`CollapsedOther`).

### 2.2 SMT Leaf Payload (Normative)

```
leaf_payload(lane) = lane_tip || le_u64(last_touch_blue_score)
leaf_hash(lane)    = H_leaf( leaf_payload(lane) )
```

`lane_id` is *not* in the preimage: it is already bound by the `lane_key` SMT path and recursively inside `lane_tip` via `H_lane_tip(...)` ([§5](#5-tip-update-hash)). See [KIP-21 §6.2](./kip-0021.md#62-leaf-value) for the rationale.

Reference: [`consensus/seq-commit/src/types.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/seq-commit/src/types.rs), [`consensus/seq-commit/src/hashing.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/seq-commit/src/hashing.rs).

## 3. Activity Leaf and Digest

### 3.1 Activity Leaf (Normative)

```
activity_leaf(tx, merge_idx) = H_activity_leaf( tx.id || le_u16(tx.version) || le_u32(merge_idx) )
```

`tx.id` is the 32-byte transaction ID; `tx.version` is the 2-byte LE-encoded transaction version. `merge_idx` is the 0-based index of `tx` in `AcceptedTxList(B)` ([KIP-21 §4.2](./kip-0021.md#42-per-lane-block-activity-digest)).

### 3.2 Activity Digest (Normative)

`activity_digest(lane, B)` is the Merkle root over the activity-leaf list for `lane` in `B`, computed with `H_seq` as the branch hash.

Merkle-root conventions used everywhere in this KIP:

- Empty list root: `ZERO_HASH`.
- Single-entry root: the entry itself (standard Merkle convention, matching `kaspa_merkle::calc_merkle_root`).
- Internal nodes: `H_seq(left, right)`. Missing right siblings at the leaf level pad with `ZERO_HASH`; padding follows the next-power-of-two rule used by `calc_merkle_root`.

The same Merkle convention is used by `MinerPayloadRoot(B)` ([§6](#6-mergeset-miner-payload-commitment)).

Reference: [`consensus/seq-commit/src/hashing.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/seq-commit/src/hashing.rs), [`crypto/merkle/src/lib.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/crypto/merkle/src/lib.rs).

## 4. Mergeset Context Hash

```
ctx_hash(B) = H_mergeset_context( le_u64(B.timestamp)
                                || le_u64(B.daa_score)
                                || le_u64(B.blue_score) )
```

`ctx_hash(B)` is committed both as part of `SeqStateRoot(B)` (as the left half of `payload_and_ctx_digest = H_seq(ctx_hash, MinerPayloadRoot)`, see [KIP-21 §6.7](./kip-0021.md#67-sequencing-state-root-normative)) and inside every touched lane-tip update in `B` ([§5](#5-tip-update-hash)). Including it on the lane-local path is what makes accepting-block clock/context available without forcing per-block global processing.

Reference: [`consensus/seq-commit/src/hashing.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/seq-commit/src/hashing.rs).

## 5. Tip Update Hash

```
TipUpdateHash(parent_ref, lane_id, activity_digest, ctx_hash)
    = H_lane_tip( parent_ref || lane_key(lane_id) || activity_digest || ctx_hash )
```

`parent_ref` is either the previous lane tip (continuing lane) or `SeqCommit(parent(B))` (reactivating lane); see [KIP-21 §5.2](./kip-0021.md#52-update-or-initialize-lane-tips).

The lane tip is *shortcut-free*: `inactivity_shortcut` is not part of its preimage. See [KIP-21 §6.6](./kip-0021.md#66-activity-root-normative) for why the shortcut sits one level above the SMT root.

Reference: [`consensus/seq-commit/src/hashing.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/seq-commit/src/hashing.rs).

## 6. Mergeset Miner Payload Commitment

For each mergeset block `X` of accepting block `B`:

```
miner_payload_hash(X) = H_miner_payload( payload_bytes(X) )

miner_payload_leaf(X) = H_miner_payload_leaf( X.hash
                                            || blue_work_encoding(X.header.blue_work)
                                            || miner_payload_hash(X) )
```

Encodings:

- `payload_bytes(X)`: the raw coinbase-tx payload bytes of `X` (the extended payload, possibly above the legacy hard limit and charged via transient mass).
- `blue_work_encoding(work)`: `le_u64(len(stripped)) || stripped`, where `stripped` is the big-endian bytes of `work` with leading zero bytes removed (empty for `work = 0`). Matches the consensus `write_blue_work` encoding; a preimage-parity test locks the two together.

Leaf ordering and root:

- `MinerPayloadLeaves(B)`: list of `miner_payload_leaf(X)` over every mergeset block `X` of `B`, in the deterministic mergeset traversal order (selected parent first, then the remaining mergeset blocks in topological order).
- `MinerPayloadRoot(B) = MerkleRoot(MinerPayloadLeaves(B))` with `H_seq` branching and the conventions from [§3.2](#32-activity-digest-normative).

Reference: [`consensus/seq-commit/src/hashing.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/seq-commit/src/hashing.rs), [`consensus/src/pipeline/virtual_processor/utxo_validation.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/src/pipeline/virtual_processor/utxo_validation.rs) (`collect_mergeset_seq_data`).

## 7. Activity Root

```
ActivityRoot(B) = H_activity_root( inactivity_shortcut(B) || ActiveLanesRoot(B) )
```

`inactivity_shortcut(B)` is defined in the [Inactivity Shortcut Spec §2.2](https://github.com/kaspanet/vprogs/blob/feat/inactivity_proofs/docs/kaspa-inactivity-shortcut-spec.md#22-inactivity_shortcut-definition). When no qualifying ancestor exists (near genesis), it is `ZERO_HASH`; otherwise it is the `SeqCommit` of the most recent in-domain non-genesis ancestor beyond the staleness boundary, with `min(F(A), F(B))` predicate for `F`-rotation handling.

Full anchors used by L2 inactivity-proof witnesses always recompose `ActivityRoot` via this wrap; this is what enforces the proof format without the guest knowing `F`. See [Inactivity Shortcut Spec §4.2](https://github.com/kaspanet/vprogs/blob/feat/inactivity_proofs/docs/kaspa-inactivity-shortcut-spec.md#42-nextanchorpath--the-per-anchor-walk-decision).

Reference: [`consensus/seq-commit/src/hashing.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/seq-commit/src/hashing.rs) (`activity_root_hash`); [`consensus/src/pipeline/virtual_processor/utxo_validation.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/src/pipeline/virtual_processor/utxo_validation.rs) (`build_seq_commit`).

## 8. Storage Layout

The SMT consensus state is split across four DB column families plus an in-memory cache layer. Each row carries an immutable per-block version so reorgs do not require destructive rewrites. Reference: [`consensus/smt-store/src/keys.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/smt-store/src/keys.rs), [`consensus/smt-store/src/values.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/smt-store/src/values.rs).

### 8.1 Reverse-Blue-Score Encoding

All time-ordered keys encode `blue_score` as `rev_blue_score = u64::MAX - blue_score` in big-endian. Lexicographic forward iteration therefore yields newest scores first, so seeks for "latest version at or below `pov_blue_score`" become a single forward scan from the synthesized seek key. See [`consensus/smt-store/src/reverse_blue_score.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/smt-store/src/reverse_blue_score.rs).

### 8.2 BranchVersionStore

Stores per-(node, block) SMT branch nodes.

```
prefix(1) | depth(1) | node_key(32) | rev_blue_score(8) | block_hash(32)   // 74-byte key
value: node serialization (32 B for Internal, 64 B for Collapsed) or empty (tombstone)
```

`prefix` is `DatabaseStorePrefixes::SmtBranchVersions`. One write per `(depth, node_key, blue_score, block_hash)`. Rows are never updated in place; deletions are explicit. Reference: [`consensus/smt-store/src/branch_version_store.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/smt-store/src/branch_version_store.rs).

### 8.3 LaneVersionStore

Stores per-(lane, block) lane-tip records.

```
prefix(1) | lane_key(32) | rev_blue_score(8) | block_hash(32)              // 73-byte key
value: lane_tip(32)
```

`prefix` is `DatabaseStorePrefixes::SmtLaneVersions`. Lane identity is the key, not the value (matches [§2.2](#22-smt-leaf-payload-normative)). Entity-ordered seeks use the 33-byte prefix `(prefix | lane_key)` for "latest version of this lane on or before `pov`" queries. Reference: [`consensus/smt-store/src/lane_version_store.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/smt-store/src/lane_version_store.rs).

### 8.4 ScoreIndex (Purge / Replay Index)

Drives oldest-first purge processing, drives the per-block expiration scan, and acts as the per-block change manifest for reverse-application during reorgs.

Two flavors of key are written:

```
// Normal incremental write:
prefix(1) | rev_blue_score(8) | kind(1) | block_hash(32)                   // 42-byte key

// IBD batched write (chunked import, prevents collisions across chunks):
prefix(1) | rev_blue_score(8) | kind(1) | block_hash(32) | batch_id_le(4)  // 46-byte key

value: max_depth(1) | lane_keys(N * 32)                                    // 1 + 32 N bytes
```

`prefix` is `DatabaseStorePrefixes::SmtScoreIndex`. `kind` is a 1-byte discriminator: `0 = LeafUpdate` (lane inserted or tip updated at this score), `1 = Structural` (lane expired by this block, or an SLO collapse / re-expansion was emitted as a side effect). Two kinds are needed so reverse application can distinguish lane-value reversion (LeafUpdate) from tree-shape reversion (Structural) when undoing a block's diff.

Within a single block's set of emitted entries (same `rev_blue_score` and `block_hash`), `kind` sorts before `block_hash` in the key, so a LeafUpdate row for the block appears before its paired Structural row. Reverse-apply iterates a block's entries in this order, undoing leaf-value changes first and tree-shape changes second, which is the apply order (Structural shapes first, LeafUpdates layered on top) reversed.

`max_depth` (1 byte) is the deepest SMT branch level the block touched. The implementation uses it to bound depth iteration when sweeping for stale branch versions whose owning blocks have themselves been pruned (so the branch deletion can stop early once it reaches a depth no block at this score touched); without it the sweep would have to scan all 256 levels per row. Reference: [`consensus/smt-store/src/score_index.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/smt-store/src/score_index.rs).

The lane-keys list enumerates exactly which lanes the block touched (per kind), so reverse application of a chain block's diff can be done without rescanning lane/branch stores.

### 8.5 SmtBlockMetadata (Per-Block Header-Side Anchor)

The per-block, header-side record committing the SMT-anchored fields of `SeqCommit(B)`. Final shape (single hard-fork target):

```
fields (little-endian within each field):
  payload_and_ctx_digest    : Hash    (32 B)
  active_lanes_count        : u64     ( 8 B)
  inactivity_shortcut_block : Hash    (32 B)
total wire size                          : 72 B

DB key: prefix(1) | block_hash(32)                                         // 33-byte key
prefix: DatabaseStorePrefixes::SmtSeqCommitMeta
```

`payload_and_ctx_digest` is the inner `H_seq(ctx_hash, MinerPayloadRoot)` from [§4](#4-mergeset-context-hash). `active_lanes_count` sizes IBD receiver allocations and is not header-committed. `inactivity_shortcut_block` is the *block-hash* internal pointer to the target whose `SeqCommit` is the committed `inactivity_shortcut` value; storing the block hash (not the seq_commit) lets the implementation reseed the forward search for future blocks without rederiving the target's full anchor. See [Inactivity Shortcut Spec §2.2](https://github.com/kaspanet/vprogs/blob/feat/inactivity_proofs/docs/kaspa-inactivity-shortcut-spec.md#22-inactivity_shortcut-definition).

`SmtBlockMetadata` is *not* part of the header pre-image; only the folded `inactivity_shortcut` value (resolved via `inactivity_shortcut_block`) is bound into `SeqCommit(B)` via `ActivityRoot(B)`.

Not to be confused with the IBD wire message `SmtMetadataMessage` ([§12.3](#123-wire-format-protobuf)): that is the on-the-wire bundle a syncing peer receives at the pruning-point handoff. The wire and the on-disk record overlap only in `payload_and_ctx_digest`; the wire additionally carries `ActiveLanesRoot(PP)` and `SeqCommit(parent(PP))` so the receiver can recompose and verify `SeqCommit(PP)`, while the receiver derives `inactivity_shortcut_block(PP)` itself from PP-side headers and reachability rather than receiving it ([§12.2](#122-receiver-path) step 3). `ActiveLanesRoot(PP)` and `SeqCommit(parent(PP))` are not part of the on-disk per-block record because the receiver, after import, can recompute them on demand.

Reference: [`consensus/src/model/stores/smt_metadata.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/src/model/stores/smt_metadata.rs).

## 9. Cache and Versioning Model

### 9.1 Aggregate

`SmtStores` aggregates the persisted side and an in-memory cache layer:

```
SmtStores {
    branch_version : DbBranchVersionStore   (DB)
    lane_version   : DbLaneVersionStore     (DB)
    score_index    : DbScoreIndex           (DB)
    branch_cache   : BranchVersionCache     (in-memory)
    lane_cache     : LaneVersionCache       (in-memory)
}
```

Reference: [`consensus/smt-store/src/cache.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/smt-store/src/cache.rs), [`consensus/smt-store/src/processor.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/smt-store/src/processor.rs).

### 9.2 Versioning

"Versioning" here means immutable per-block snapshots, not content addressing. Each block that touches lanes appends new entries to `branch_version`, `lane_version`, and `score_index`, keyed by `(entity, blue_score, block_hash)`. Reads specify a target `pov_blue_score` and an `is_canonical(block_hash) -> bool` predicate; the seek finds the latest entry in `[pov - F, pov]` whose `block_hash` is canonical from the caller's POV.

`is_canonical(block_hash)` returns true iff `block_hash` is a selected-parent-chain ancestor of (or equal to) the caller's POV block. For the per-block processing pipeline ([§10](#10-per-block-processing-pipeline)), the POV is the current block being validated; for a peer-served read, the POV is the requested anchor. The predicate is resolved against the reachability service the consensus layer already maintains.

See `SmtReadBounds` in [`consensus/smt-store/src/processor.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/smt-store/src/processor.rs).

### 9.3 Newest-Suffix Cache Invariant (Normative for Cache Correctness)

For every entity (branch slot or lane), the cached set of versions is a *score-ordered suffix* of that entity's persisted write history. Equivalently: if the cache holds version `V_cache` for entity `E`, every newer version written through the incremental path is also in the cache.

This invariant lets cache hits be authoritative without DB fallback. It is preserved by three rules:

1. **Write-through on the incremental path.** Every `flush()` that persists a branch or lane version to DB also inserts it into the cache.
2. **Score-ordered eviction.** Insertion and eviction both remove lowest-score entries first globally; evicted entries are always a prefix per entity.
3. **Canonical tie-break.** Within a score bucket, eviction reverses `block_hash` ordering so the lowest-`block_hash` (first canonical-candidate) entry is kept longest.

IBD streaming import bypasses the caches and writes only to DB. The bypass is safe because import is preceded by `SmtStores::clear_all()`, which empties both DB and cache, so no stale cached entries can contradict the imported state.

Reference: [`consensus/smt-store/src/cache.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/smt-store/src/cache.rs).

### 9.4 Lane-Tip Read Window (Normative)

Every consensus read of an existing lane tip uses the half-open window `[pov.blue_score - F, pov.blue_score]`, regardless of whether the lookup originates in the current block, its selected parent, or a peer query. `F` is the finality-depth parameter; lanes whose latest version sits below `pov - F` are *invisible* (they will be expired by the next pass over the score index).

This window is intentionally a function of the *current* block's POV, not of the parent block's state: a lane that was visible to the parent but whose blue-score gap to the POV has crossed `F` reads as absent, which is what produces a reset (`parent_ref = SeqCommit(parent(B))`) on the same-block reactivation case. See `SmtReadBounds::for_pov` in [`consensus/smt-store/src/processor.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/smt-store/src/processor.rs).

### 9.5 Reorg Handling

`MaybeFork<T>` wraps a value with the `(blue_score, block_hash)` it came from, forcing the caller to verify canonicality before trusting the value. Long historical scans use `ReacquiringRawIterator`, which periodically drops and reopens RocksDB iterators (every 16 384 steps or every 1 s, whichever first) to release DB pins so a slow scan never starves writers. References: [`consensus/smt-store/src/maybe_fork.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/smt-store/src/maybe_fork.rs), [`consensus/smt-store/src/reacquire_iter.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/smt-store/src/reacquire_iter.rs).

## 10. Per-Block Processing Pipeline

### 10.1 Pipeline Stages

For each chain block `B`, the virtual processor performs the following stages, in order. Reference: [`consensus/src/pipeline/virtual_processor/utxo_validation.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/src/pipeline/virtual_processor/utxo_validation.rs) (`collect_mergeset_seq_data`, `resolve_lane_updates`, `build_seq_commit`), [`consensus/smt-store/src/processor.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/smt-store/src/processor.rs).

1. **Collect** per-lane activity leaves and miner-payload leaves from the mergeset (`collect_mergeset_seq_data`). Walks the mergeset in canonical order, tracks a global `merge_idx` across all accepted txs, groups activity leaves by `tx.subnetwork_id`, and emits one miner-payload leaf per mergeset block.
2. **Resolve** lane updates against existing tips (`resolve_lane_updates`). For each touched lane, look up the latest version in `[pov - F, pov]` filtered by `is_smt_canonical`. If absent, the lane reactivates and `parent_ref = SeqCommit(parent(B))`; otherwise `parent_ref` is the looked-up tip. Compute `lane_tip_next(...)`.
3. **Expire** stale lanes (`expire_stale_lanes`). Scan the score index in the half-open interval `[parent.bs - F, current.bs - F)` for lanes with no newer canonical version and emit Structural score-index entries for them.
4. **Build** the new SMT root (`SmtProcessor::build`). Each touched lane produces a leaf via `smt_leaf_hash(SmtLeafInput { lane_tip, blue_score })`. `compute_root_update` walks the SMT bottom-up against a `VersionedBranchReader` (the read-side adapter over `SmtStores` at `SmtReadBounds`), producing a fresh root plus `SmtBuild` containing branch deltas, lane changes, and score-index manifest.
5. **Wrap and chain.** `build_seq_commit` wraps the new `ActiveLanesRoot` with `inactivity_shortcut(B)` to form `ActivityRoot(B)`, combines it with `H_seq(ctx_hash, MinerPayloadRoot)` to form `SeqStateRoot(B)`, then chains under `SeqCommit(parent(B))` to form `SeqCommit(B)`.
6. **Persist** atomically. `SmtBuild::flush(write_batch)` writes branch versions, lane versions, score-index entries (LeafUpdate + Structural + SLO promotions), and updates both caches. Header-side `SmtBlockMetadata` ([§8.5](#85-smtblockmetadata-per-block-header-side-anchor)) is written by the consensus layer in the same `WriteBatch`.

### 10.2 SLO Promotion and Score-Index Manifest

When a touched lane's update collapses a subtree into a single leaf (or splits a collapsed subtree back into branches), the affected node keys are recorded in the same score-index entry's lane-key list and tagged as `Structural`. This makes the per-block diff fully reversible: reverse application can reconstruct the previous shape from the same manifest without re-deriving the SMT.

### 10.3 Atomicity

Branch versions, lane versions, score-index entries, lane/branch cache inserts, and the `SmtBlockMetadata` write all happen in a single `WriteBatch` committed by the consensus layer. Cache inserts happen in process memory inside `flush()`; they are *not* rolled back if the `WriteBatch` itself fails, but the consensus layer treats batch failure as fatal, so this race cannot be observed in practice.

## 11. Inactivity Shortcut Block Pointer

`SmtBlockMetadata.inactivity_shortcut_block` stores the *block hash* of the target whose `SeqCommit` is the committed `inactivity_shortcut(B)` value.

### 11.1 Pointer → Committed Value Fold (Normative)

```
inactivity_shortcut(B) = ZERO_HASH                     if SmtBlockMetadata(B).inactivity_shortcut_block == genesis.hash
                       = SeqCommit(target)             otherwise
where target = SmtBlockMetadata(B).inactivity_shortcut_block
```

`genesis.hash` is used as the in-band sentinel for the near-genesis case: when no qualifying ancestor exists yet, the implementation stores `genesis.hash` as the pointer so the forward-walk search always has a valid `bs <= target_bs` cascade seed; the fold rule then collapses it to the `ZERO_HASH` committed value before binding into `ActivityRoot(B)`.

For non-sentinel pointers, the target's `SeqCommit` is looked up from the per-block header (`target.accepted_id_merkle_root`).

### 11.2 Computing the Pointer (Implementation)

The pointer itself is computed by `compute_inactivity_shortcut_block` in [`consensus/src/pipeline/virtual_processor/processor.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/src/pipeline/virtual_processor/processor.rs), using the parent block's pointer as a cascade seed and walking forward through header metadata under the predicate `A.blue_score + min(F(A), F(B)) < B.blue_score` (see [Inactivity Shortcut Spec §2.2](https://github.com/kaspanet/vprogs/blob/feat/inactivity_proofs/docs/kaspa-inactivity-shortcut-spec.md#22-inactivity_shortcut-definition) for the normative predicate definition).

Two fast paths in the implementation:

- **Coinbase-lane fast path.** Every chain block touches the coinbase lane, so `lane_version[COINBASE_LANE_KEY]` for the relevant `(pov - F)` window is dense; the shortcut search reads it directly to identify the candidate boundary block.
- **Parent's recorded shortcut as seed.** The forward walk starts from the parent's `inactivity_shortcut_block` rather than re-searching from scratch, amortizing the search across the chain.

Near-genesis behavior follows directly from the fold rule of §11.1: the implementation seeds the internal pointer with `genesis.hash` for the initial range and the fold then commits `ZERO_HASH`. No special-case code path on the consumer side.

## 12. IBD Bootstrap

### 12.1 Sender Path

The sender exports the active-lane set of the pruning point `PP` as a stream of `(lane_key, lane_tip_hash, blue_score)` triples (`ImportLane`), in chunks of `SMT_CHUNK_SIZE = 4096` lanes (≈ 288 KiB per chunk at 72 B per entry, plus protobuf framing overhead). The set is constructed by iterating `lane_version.iter_all_canonical_owned()` with `max_blue_score = PP.blue_score`, so any post-`PP` updates are excluded. See `open_pruning_point_smt_lane_stream` in [`consensus/smt-store/src/lane_version_store.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/smt-store/src/lane_version_store.rs).

### 12.2 Receiver Path

On the receiver:

1. `SmtStores::clear_all()` clears DB and cache for SMT-related column families.
2. `SmtMetadataMessage` (header-only) is received: three 32-byte hashes packed into a single 96-byte `bytes` field in the order `ActiveLanesRoot(PP) || payload_and_ctx_digest(PP) || SeqCommit(parent(PP))`, plus `active_lanes_count` as `u64`. `inactivity_shortcut_block(PP)` is *not* on the wire.
3. The receiver derives `inactivity_shortcut_block(PP)` locally from the already-validated PP-side headers and reachability via `async_inactivity_shortcut_block_for_pov(PP)`, then folds it through the rule in [§11.1](#111-pointer--committed-value-fold-normative) to obtain the committed `inactivity_shortcut(PP)` value. Doing the derivation locally is safe at the PP boundary because the pruning-proof chain (and therefore the headers up to and including `PP`) has been validated before this phase.
4. `SmtLaneChunkMessage`s are streamed in. The receiver hashes leaves in parallel via rayon, then feeds them into `StreamingSmtBuilder` (no_std-friendly) over a `DbSink` writer. The builder reconstructs branch nodes bottom-up; the sink writes them at the lane's own blue score (matching the sender's per-lane versioning) and records seal events. When all lanes contributing to a score-index entry are sealed, the entry is flushed.
5. The receiver computes `ActiveLanesRoot` from the leaf stream via `SmtStores::recompute_lanes_root_from_leaf_stream()` and checks it equals the `ActiveLanesRoot(PP)` claimed by the metadata.
6. The receiver composes `ActivityRoot(PP) = H_activity_root(inactivity_shortcut(PP), ActiveLanesRoot(PP))`, then `SeqStateRoot(PP)`, and finally checks `seq_commit(parent_seq_commit, SeqStateRoot(PP)) == PP.accepted_id_merkle_root`.

Once verified, normal incremental block processing resumes from `PP`.

References: [`consensus/smt-store/src/streaming_import/mod.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/smt-store/src/streaming_import/mod.rs), [`consensus/smt-store/src/streaming_import/db_sink.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/smt-store/src/streaming_import/db_sink.rs), [`protocol/flows/src/v10/request_pruning_point_smt_state.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/protocol/flows/src/v10/request_pruning_point_smt_state.rs), [`protocol/flows/src/ibd/streams.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/protocol/flows/src/ibd/streams.rs).

### 12.3 Wire Format (Protobuf)

The IBD-side proto messages (full schema in [`protocol/p2p/proto/p2p.proto`](https://github.com/kaspanet/rusty-kaspa/blob/master/protocol/p2p/proto/p2p.proto)):

- `SmtMetadataMessage` (sent first, single message): three 32-byte hashes packed into a single 96-byte `bytes` field in the order `lanes_root || payload_and_ctx_digest || parent_seq_commit`, plus `active_lanes_count` as `u64`. `inactivity_shortcut_block(PP)` is *not* shipped; the receiver derives it locally from PP-side headers and reachability (see [§12.2](#122-receiver-path) step 3).
- `SmtLaneChunkMessage` (repeated): a protobuf-repeated list of `SmtLaneEntry` records (`lane_key(32) | lane_tip_hash(32) | blue_score(u64)`), sized at `SMT_CHUNK_SIZE` per message.

Flow control: the sender pauses after every 10th chunk for an explicit "next batch please" ack from the receiver, so a slow receiver does not stall the sender's write queue.

### 12.4 Pruning-Proof Chain Segment

When validating the pruning-proof chain segment in IBD, the lower-bound context for `seq_commit` recomputation is `pp.sp.blue_score` (the pruning point's *selected parent's* blue score). Using `pp.blue_score` would exclude the chain block exactly at the activity-window boundary and miss the inactivity-shortcut anchor. Reference: [`consensus/src/processes/pruning_proof/apply.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/src/processes/pruning_proof/apply.rs).

## 13. Invariants

A summary of the invariants load-bearing for correctness; each is enforced by the code referenced inline or above.

- **(I1) Lane-tip read window** ([§9.4](#94-lane-tip-read-window-normative)). Every consensus read of an existing lane tip uses `[pov.bs - F, pov.bs]`. Same-block reactivation (`pov.bs - F` boundary crossing) yields `is_new = true` and a reset via `parent_ref = SeqCommit(parent(B))`. The matching expire in `expire_stale_lanes` cancels the `is_new = true` arithmetic for `active_lanes_count`.
- **(I2) Newest-suffix cache invariant** ([§9.3](#93-newest-suffix-cache-invariant-normative-for-cache-correctness)). For every entity, the cached set is a score-ordered suffix of the entity's persisted write history. Cache hits are authoritative; no DB fallback needed on hit.
- **(I3) Per-block diff reversibility** ([§10.2](#102-slo-promotion-and-score-index-manifest)). Branch versions, lane versions, score-index entries (including SLO promotion structural entries) for one block are written in one `WriteBatch`. Reverse application reads the score-index manifest and walks branch/lane versions back to the prior canonical state.
- **(I4) Shortcut-free lane tips** ([§5](#5-tip-update-hash)). `H_lane_tip` preimage does not include `inactivity_shortcut`. Two-anchor lane proofs therefore stay valid across changes in `inactivity_shortcut(B)` between anchors.
- **(I5) Activity-root wrap order** ([§7](#7-activity-root)). `H_activity_root(inactivity_shortcut, ActiveLanesRoot)` (shortcut first, lanes second). Order is consensus-fixed.
- **(I6) Seq-commit recurrence order** ([KIP-21 §6.8](./kip-0021.md#68-sequencing-commitment-recurrence-normative)). `seq_commit(parent_seq_commit, state_root)` (parent first, state second).
- **(I7) IBD-import / live-state disjointness** ([§9.3](#93-newest-suffix-cache-invariant-normative-for-cache-correctness), [§12.2](#122-receiver-path)). IBD streaming import is preceded by `clear_all()` and bypasses caches. No live read can observe a partial import because the consensus layer holds the relevant locks across the import phase.
- **(I8) Reverse-blue-score key encoding** ([§8.1](#81-reverse-blue-score-encoding)). All time-ordered DB keys use `u64::MAX - blue_score` in big-endian. A forward seek for `(entity, target_bs)` yields the latest version on or below `target_bs` in one read.
- **(I9) Score-index sort discipline** ([§8.4](#84-scoreindex-purge--replay-index)). Within a score bucket, `kind` sorts before `block_hash`, putting LeafUpdate ahead of Structural so reverse-apply can revert in the correct order without explicit sequencing.
- **(I10) Sender/receiver versioning parity** ([§12.1](#121-sender-path), [§12.2](#122-receiver-path)). The sender exports lanes at `max_blue_score = PP.blue_score` and the receiver writes versioned branch nodes at each lane's *own* blue score, matching what the live sender's DB held at `PP`. Reorg safety follows from this parity.

## 14. Optional Persistent Witness Store

A unified, content-addressed commitment-node store keyed by `node_hash`:

```
node_hash -> (left_child_hash, right_child_hash)
```

covers both:

- `H_seq` nodes (sequencing commitment, state-root, and activity-root composition), and
- SMT internal nodes (`H_node` branches and `H_collapsed_node` collapsed leaves) under `ActivityRoot(B)`.

Empty subtree hashes (`EMPTY_i`, [§2.1](#21-slo-single-leaf-optimization-normative)) are computable from this spec; the store only retains non-empty nodes. Collapsed nodes are stored as a single `(lane_key, leaf_hash)` entry rather than as a chain of 256 internal nodes; implementations can either record a discriminator or fold collapsed nodes into the same `(left, right)` schema by treating `lane_key` and `leaf_hash` as the two 32-byte halves under the `H_collapsed_node` domain.

Reconstructing a lane witness from a header anchor `accepted_id_merkle_root = SeqCommit(B)` follows a fixed prefix path:

- right child to `SeqStateRoot(B)`;
- left child to `ActivityRoot(B)`;
- right child of `ActivityRoot(B)` to `ActiveLanesRoot(B)`;
- then the key-driven SMT path on `lane_key(lane)`, collecting siblings (and stopping early at a collapsed terminal where applicable, [§2.1](#21-slo-single-leaf-optimization-normative)).

Reference-counted cleanup:

- maintain `refcount[node_hash]` alongside `node_hash -> (left, right)`;
- on first insertion of an entry, increment `refcount[left]` and `refcount[right]`;
- treat each retained header anchor as one additional reference to its root (increment on retention, decrement on prune), even if the root node already exists;
- when a refcount hits zero, delete the node and recursively decrement its children.

This layer is optional and does not replace reorg diffs for hot-path consensus maintenance.

## 15. Source-File Index

A flat index of the files this document references, for navigating the reference implementation:

- Hash domains: [`crypto/hashes/src/hashers.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/crypto/hashes/src/hashers.rs).
- Pure hash + types: [`consensus/seq-commit/src/`](https://github.com/kaspanet/rusty-kaspa/tree/master/consensus/seq-commit/src) (`hashing.rs`, `types.rs`, `verify.rs`).
- SMT core: [`crypto/smt/src/`](https://github.com/kaspanet/rusty-kaspa/tree/master/crypto/smt/src) (`tree.rs`, `proof.rs`, `store.rs`, `streaming.rs`); empty-hash generation in [`crypto/smt/build.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/crypto/smt/build.rs).
- SMT storage: [`consensus/smt-store/src/`](https://github.com/kaspanet/rusty-kaspa/tree/master/consensus/smt-store/src) (`keys.rs`, `values.rs`, `branch_version_store.rs`, `lane_version_store.rs`, `score_index.rs`, `cache.rs`, `processor.rs`, `maybe_fork.rs`, `reacquire_iter.rs`, `reverse_blue_score.rs`, `streaming_import/`).
- Per-block metadata: [`consensus/src/model/stores/smt_metadata.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/src/model/stores/smt_metadata.rs).
- Block processing: [`consensus/src/pipeline/virtual_processor/utxo_validation.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/src/pipeline/virtual_processor/utxo_validation.rs), [`consensus/src/pipeline/virtual_processor/processor.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/consensus/src/pipeline/virtual_processor/processor.rs).
- IBD flow: [`protocol/flows/src/v10/request_pruning_point_smt_state.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/protocol/flows/src/v10/request_pruning_point_smt_state.rs), [`protocol/flows/src/ibd/streams.rs`](https://github.com/kaspanet/rusty-kaspa/blob/master/protocol/flows/src/ibd/streams.rs).
- Wire format: [`protocol/p2p/proto/p2p.proto`](https://github.com/kaspanet/rusty-kaspa/blob/master/protocol/p2p/proto/p2p.proto), [`protocol/p2p/proto/messages.proto`](https://github.com/kaspanet/rusty-kaspa/blob/master/protocol/p2p/proto/messages.proto).
- Pruning proof: [`consensus/src/processes/pruning_proof/`](https://github.com/kaspanet/rusty-kaspa/tree/master/consensus/src/processes/pruning_proof).
