# High-Performance Ledger State Machine Design in TiKV

## 1. Overview

This document outlines the design for a high-performance replicated state machine implemented on top of TiKV. The goal is to build a "Ledger" database capable of strongly-consistent (linearizable) operations such as `Get`, `Add`, and `Subtract` on numeric values (balances).

TiKV is built on RocksDB and Raft (via `raftstore`), providing a robust distributed key-value store with ACID transactional guarantees (MVCC and Percolator). However, the default transactional layer introduces overhead that might not be necessary for simple, commutative ledger operations.

To achieve maximum performance for this specific state machine, the design proposes bypassing the standard transaction/MVCC stack and integrating directly with the Raft layer as a specialized state machine.

## 2. Goals and Requirements

*   **Operations:** `Get` (strongly consistent), `Add`, and `Subtract`.
*   **Semantics:** Balances cannot be negative. A `Subtract` operation that results in a negative balance must fail.
*   **Consistency:** Linearizable (Strong Consistency). Every read must reflect the latest committed write.
*   **Performance:** High throughput and low latency, minimizing the overhead of the existing MVCC and transaction layers.

## 3. Architecture Overview

TiKV's architecture typically flows like this for a standard request:
`gRPC Service -> Transaction Layer (Percolator/MVCC) -> Raftstore -> RocksDB`

For the high-performance Ledger state machine, the proposed architecture is:
`gRPC Ledger Service -> (Bypass MVCC) -> Raftstore (Custom Command) -> Ledger Apply Delegate -> RocksDB`

By bypassing MVCC, we eliminate the overhead of timestamp allocation (from PD), lock resolution, and multi-version storage management. We treat the ledger values as single-version data managed directly by the Raft state machine.

## 4. Components

### 4.1. gRPC Service (`proto/ledger.proto`)

A new gRPC service defined in `ledger.proto` exposes the `Get`, `Add`, and `Subtract` endpoints. The TiKV `server` component will be extended to implement and register this service.

### 4.2. Raftstore Integration

`raftstore` is the core component that manages Raft groups (Regions). To support the Ledger state machine, we must modify `raftstore` to handle custom command types.

1.  **Command Routing:** Currently, `raftstore` processes standard key-value requests (`Put`, `Delete`, `Scan`, etc.). We need to introduce a new request type or extend the existing `RaftCmdRequest` to encapsulate Ledger commands (`AddRequest`, `SubtractRequest`).
2.  **Proposing Commands:** The Ledger gRPC service will convert incoming requests into Raft commands and propose them to the appropriate Raft group (Region) based on the key's hash or range.
3.  **Strong Consistency for Reads:** `Get` operations must be strongly consistent. We will use TiKV's `ReadIndex` mechanism. When a `Get` request is received, the leader proposes a `ReadIndex` request to the Raft group. Once a quorum acknowledges the read index, the leader can safely serve the read from its local state, guaranteeing linearizability without writing a Raft log entry.

### 4.3. Ledger Apply Delegate

When a Raft command is committed, the `ApplyFsm` (Apply Finite State Machine) in `raftstore` processes it. We need to implement a specialized "Apply Delegate" for the Ledger commands.

1.  **State Representation:** Ledger values (uint64 balances) will be stored directly in RocksDB using a specific key prefix (e.g., `zLedger_` + `key`) to isolate them from standard MVCC data. The values will be 8-byte integers (Little Endian).
2.  **Applying Add:**
    *   Read the current value from RocksDB. If it doesn't exist, assume 0.
    *   Add the requested amount (checking for overflow).
    *   Write the new value to RocksDB within the current apply batch.
3.  **Applying Subtract:**
    *   Read the current value from RocksDB. If it doesn't exist, assume 0.
    *   Check if the current balance is sufficient. If `current < amount`, the operation fails (this logic needs to be deterministic to avoid state divergence).
    *   If sufficient, subtract the amount and write the new value to RocksDB.

### 4.4. Handling State Machine Failures (Determinism)

A crucial aspect of Raft state machines is determinism. If a `Subtract` operation fails due to insufficient funds, this failure must be consistent across all replicas.

*   **Deterministic Evaluation:** The validation logic (checking if balance >= amount) must happen *during the apply phase*, not before proposing. The command proposed to Raft is simply "Attempt to subtract X from Key Y".
*   **Result Reporting:** The `ApplyDelegate` must communicate the result of the operation (success or failure, and the new balance) back to the client. This is typically done via a callback channel established when the command was proposed.

## 5. Proposed Implementation Steps

1.  **Define Protobuf:** (Completed) Create `ledger.proto` defining the gRPC interface.
2.  **Extend `kvproto`:** (To do in a real implementation) Add `LedgerRequest` and `LedgerResponse` to the internal Raft command definitions (`raft_cmdpb.proto` in the `kvproto` repository) to allow routing these commands through `raftstore`.
3.  **Implement Ledger gRPC Service:** In `src/server/service`, create `ledger.rs` implementing the `Ledger` gRPC service. This service will translate protobuf requests into internal Raft commands and interact with the `RaftRouter`.
4.  **Modify `raftstore` Command Handling:**
    *   Update `src/raftstore/store/fsm/peer.rs` to handle proposing Ledger commands.
    *   Implement the `ReadIndex` logic for `Get` requests to ensure strong consistency.
5.  **Implement the Apply Logic:**
    *   In `src/raftstore/store/fsm/apply.rs` (or a dedicated module), implement the logic to process committed Ledger commands.
    *   This logic will directly read from and write to the underlying RocksDB engine, bypassing the MVCC layer.
    *   Ensure proper handling of the `Subtract` validation logic (failing if balance is insufficient) deterministically during the apply phase.
6.  **Testing:**
    *   Add unit tests for the apply logic.
    *   Add integration tests in `tests/` to verify linearizability, failover handling, and the "insufficient funds" constraint.

## 6. Performance Considerations

*   **Bypassing MVCC:** The primary performance gain comes from bypassing the MVCC layer. We save on timestamp allocation from PD, lock management, and storing multiple versions of the same key.
*   **Batching:** TiKV's `raftstore` heavily relies on batching for both Raft proposals and RocksDB writes. The Ledger commands will naturally benefit from this existing infrastructure.
*   **ReadIndex for Gets:** Using `ReadIndex` is crucial for performance as it avoids the overhead of writing a Raft log entry for every read, while still maintaining strong consistency.

## 7. Future Enhancements

*   **Coprocessor Integration:** If complex analytical queries over the ledger data are required, the Coprocessor framework could be extended to understand the single-version Ledger format.
*   **Snapshot Management:** Ensure TiKV's region snapshotting mechanism correctly captures the Ledger state alongside standard MVCC data.