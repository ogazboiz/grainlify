## Description
Closes #603

Implements auto-compaction and pruning of historical state for the `bounty_escrow` and `program_escrow` smart contracts. This reduces the blockchain storage footprint and bounds iteration over unbounded indexes to minimize gas costs. 

### Implementation Details:
1. **Bounty Escrow (`prune_bounties`)**:
   - Allows the `Admin` to prune an array of `bounty_ids`.
   - Checks that each provided escrow is in a terminal state (`Released` or `Refunded`).
   - Removes escrow data (`Escrow`, `Metadata`, `PendingClaim`, `RefundApproval`, `ReleaseApproval`) from persistent storage.
   - Cleans up the `EscrowIndex` and `DepositorIndex`.
   - Emits a `BountyPruned` event.

2. **Program Escrow (`prune_program`)**:
   - Allows the `Admin` to prune a specific `program_id`.
   - Safely checks for both `PROGRAM_DATA` (single-program mode) and `DataKey::Program(id)` (multi-program mode).
   - Validates that the program's `remaining_balance == 0`.
   - Removes all relevant state keys (`NextScheduleId`, `ReleaseHistory`, `Schedules`, `MultisigConfig`, etc.).
   - Emits a `ProgramPrunedEvent`.

### Operational Usage & Safeguards:
- **Off-chain Age Tracking**: Pruning does not rely on on-chain timestamp tracking for performance reasons. Off-chain infrastructure should track the `BountyCompleted` or `Payout` events, determine when an escrow/program is sufficiently "old" (e.g., 90 days), and proactively call the `prune_*` admin functions.
- **Auditability**: We emit explicit `BountyPruned` and `ProgramPruned` events immediately upon data deletion. Indexers and off-chain data warehouses should consume these events to maintain a complete historical ledger.
- **State Protection**: The contracts maintain strict assertions ensuring only `Admin` can prune and that no active funds are accidentally wiped. If an escrow/program is not in its terminal state with zero remaining balance, the transaction will panic with an `InvalidState` contract error.

### Verification
- Fully covered with new automated unit tests evaluating positive paths and edge-cases (unauthorized accesses, premature pruning).
- Local verification with `cargo test`, `cargo fmt` complete.
