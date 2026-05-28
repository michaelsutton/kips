# KIP-21 Seqcommit Accessor

Status: Companion document.

This document describes the script-facing accessor for KIP-21 sequencing
commitments. The consensus commitment itself is specified in
[KIP-21](../kip-0021.md).

## Background

Based zk applications use L1 ordering as the source of truth for the data they
must execute. A typical state-transition transaction spends an old
state-commitment UTXO, proves an application transition, and creates a new
state-commitment UTXO. The script or covenant logic must therefore be able to
check that the claimed new state is anchored to a real L1 chain-block
sequencing commitment.

`OpChainblockSeqCommit` is that script-facing anchor. It lets a script read the
seqcommit of a recent selected-parent-chain block and compare it with the public
state-transition data committed by the application proof. This is the successor
of the `OpChainBlockHistoryRoot` idea described in the based-apps background
posts ([based zk-rollups](https://research.kas.pa/t/on-the-design-of-based-zk-rollups-over-kaspas-utxo-based-dag-consensus/208),
[canonical bridge](https://research.kas.pa/t/l1-l2-canonical-bridge-entry-exit-mechanism/258)).

## Opcode Mechanics

`OpChainblockSeqCommit` consumes one 32-byte block hash from the stack and
pushes the 32-byte sequencing commitment of that chain block:

```
input:  block_hash
output: SeqCommit(block_hash)
```

The returned value is the target header's `accepted_id_merkle_root`, interpreted
post-KIP-21 as `SeqCommit(target)`.

The opcode is evaluated from a selected-parent point of view. For a transaction
validated while constructing or validating a chain block `B`, that point of view
is `selected_parent(B)`. The opcode succeeds only if the requested block:

1. is known to the node from this point of view;
2. is on the selected-parent chain of the point-of-view block;
3. is within the seqcommit access threshold; and
4. already has KIP-21 `accepted_id_merkle_root` semantics.

If any condition fails, script validation fails.

## Access Threshold

Let `P` be the selected-parent point of view and let `X` be the requested chain
block. The accessor admits `X` only when:

```
X.blue_score + F > P.blue_score
```

Equivalently, scripts can access seqcommits from the last `F` blue-score units
only. For the deployed parameters, `F` is the finality depth, corresponding to
about 12 hours of target block time.

This restriction keeps script-visible header access strictly below the pruning
period. Among other reasons, this bound determines the selected-parent-chain
header segment that a pruning-point syncer must provide below the pruning point:
the segment must cover the possible seqcommit accessor range needed at the
boundary.

## Lagging Provers

Based applications whose provers stop producing updates for longer than the
access threshold cannot resume by directly asking L1 script for an old anchor.
They must first catch up to a recent anchor that is still within the accessible
range.

This does not prevent proving longer historical spans. A prover can prove an
arbitrarily long span as long as it has the required data and witnesses; the
threshold only restricts which historical seqcommit values can be read directly
by script at validation time. Long-span proving and inactivity shortcuts are
covered in [Proving Spec](proving-spec.md).

## Reorg Behavior

The opcode never gives access to an arbitrary DAG block. It only gives access to
headers on the selected-parent chain of the current point of view.

If a transaction is reorged and later validated through a different
selected-parent chain, the same opcode call is re-evaluated from the new point of
view. If the requested block is no longer on that selected-parent chain, or is
outside the access threshold, the transaction fails under the new chain. The UTXO
state then rolls back to the last shared selected-parent-chain block in the usual
way.

This makes seqcommit access deterministic under DAG reorgs: the commitment read
by script is always relative to the selected-parent chain that is actually being
validated.
