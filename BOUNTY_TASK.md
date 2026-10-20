# Bounty Objective: [BOUNTY] High: Unsegregated fee accounting in bridge_vault allows user withdrawals to drain validator rewards and permanently deadlock unbonding stake exit
Target: paraloom-labs/paraloom-core Issue #1138
URL: https://github.com/paraloom-labs/paraloom-core/issues/1138

## Issue Description
### Summary
- **Severity:** High ($400 USDC — Stage 1 On-Chain Bridge & Staking Invariant)
- **Affected Files:** `programs/paraloom/src/lib.rs` (lines 440–474 in `transact`, lines 1007–1050 in `claim_rewards`, lines 1673–1746 in `withdraw_unbonded_stake`)
- **Payout Address:** `3HLDkipdKmJKGjbgCD8moo3GyhEY4DWdZ7JAgSpMysHp`

---

### Vulnerability Details

#### 1. The Vulnerability Mechanism
In `programs/paraloom/src/lib.rs`, when a validator settles a native SOL withdrawal via `transact` (lines 425–466), external funds are moved out of `bridge_vault`:
```rust
let payout = gross - fee;
let vault_balance = ctx.accounts.bridge_vault.lamports();
let rent_floor = Rent::get()?.minimum_balance(0);
require!(
    vault_balance >= payout.saturating_add(rent_floor),
    BridgeError::InsufficientFunds
);
// payout transferred to recipient
validator_account.pending_rewards = validator_account
    .pending_rewards
    .checked_add(fee)
    .ok_or(BridgeError::InvalidAmount)?;
```
Notice that:
1. `fee` is credited to `validator_account.pending_rewards`, but the corresponding lamports are **never segregated into a dedicated rewards vault or transferred inline**; they remain co-mingled in the shared `bridge_vault`.
2. On every subsequent withdrawal (`transact`, line 444), the solvency check only verifies that `vault_balance >= payout.saturating_add(rent_floor)`. It **does not reserve accrued validator pending rewards**. As a result, subsequent user withdrawals can completely drain the lamports representing validators' accumulated fees.

#### 2. The Exit Deadlock (Permanent Capital Lock)
When a validator unregisters and waits out `UNBONDING_SLOTS`, its exit instruction is `withdraw_unbonded_stake` (lines 1673–1746).
Line 1694 strictly enforces:
```rust
require!(
    validator_account.pending_rewards == 0,
    BridgeError::PendingRewardsUnclaimed
);
```
To satisfy this constraint, the validator is forced to call `claim_rewards` (lines 1007–1050) first.
However, `claim_rewards` attempts to transfer `reward_amount` directly from `bridge_vault` (lines 1024–1034). If user withdrawals have drained `bridge_vault` below `reward_amount + rent_floor`:
- `claim_rewards` fails with runtime `InsufficientFunds` / `InsufficientFundsForRent`.
- `validator_account.pending_rewards` remains `> 0`.
- `withdraw_unbonded_stake` is permanently blocked (`PendingRewardsUnclaimed`).
- Anchor's `close = validator` constraint is never reached.

**Consequence:** The validator's entire native SOL stake (`unbonding_amount` >= 100 SOL) and dual-stake SPL collateral (`token_unbonding_amount`) become **permanently frozen inside the contract with zero operational recourse**.

---

### Minimal Fix
1. In `BridgeState`, maintain `total_pending_rewards: u64`, incremented on `transact` fees and decremented on `claim_rewards`.
2. In `transact` (line 444), ensure user withdrawals cannot consume accrued validator fees:
```diff
- require!(
-     vault_balance >= payout.saturating_add(rent_floor),
-     BridgeError::InsufficientFunds
- );
+ require!(
+     vault_balance >= payout.saturating_add(rent_floor).saturating_add(bridge_state.total_pending_rewards),
+     BridgeError::InsufficientFunds
+ );
```
3. In `claim_rewards` (lines 1013–1020), add an explicit rent floor check before CPI transfer.


## Instructions for Kuro (Autonomous Solver):
1. Use skills: `systematic-debugging` and `code-review-and-quality`.
2. Inspect the repository structure, locate relevant files, and understand the bug/feature.
3. Apply the minimal, robust fix or implementation.
4. Run any existing test suites or linters in the workspace.
5. Ensure the fix adheres to the project's existing code style.
6. When done, output a summary of files modified and verification results.
