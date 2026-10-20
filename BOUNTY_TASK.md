# Bounty Objective: [BOUNTY] Medium: Discovery wallet claim needs no proof of ownership — impersonating any registered validator burns the per-nullifier co-sign budget and freezes a targeted settlement
Target: paraloom-labs/paraloom-core Issue #1145
URL: https://github.com/paraloom-labs/paraloom-core/issues/1145

## Issue Description
**Component:** `src/node/` (Discovery handling, co-sign ingress) + `src/consensus/transact.rs` (validator registration/admission). In scope (Stage 1: `src/consensus/`, settlement/co-sign path of `src/node/`). Checked `docs/security-log.md` — not listed (this is the residual gap of the #627 fix: registration now checks the claimed wallet is in the on-chain set, but never that the claimant owns it).

**Summary:** When a peer sends `Message::Discovery`, a node registers `node_info.wallet_pubkey` — a self-asserted string — via `register_validator_with_wallet` with no signature challenge or any other proof that the peer controls that Solana wallet. Admission (`src/consensus/transact.rs`) only checks the claimed wallet is in the on-chain validator set. All downstream authorization (`source_is_onchain_validator`, gating `handle_cosign_request` and verification requests) trusts this NodeId→wallet mapping.

**Attack:**
1. Join the mesh with any libp2p key; send Discovery claiming `node_type = ResourceProvider` and `wallet_pubkey` = a real registered validator's wallet. Every honest node now treats you as that validator. No stake, no wallet key needed.
2. Copy a victim settlement's public parameters from gossip (recipient, nullifiers, output commitments, root, ext_amount, mint).
3. Send each honest validator 4 crafted co-sign requests matching its `verified_transacts` cache (the proof bytes are not compared). Each accepted request increments the per-`(nullifier, authority)` counter; at `MAX_COSIGNS_PER_SETTLEMENT = 4` the budget for both input nullifiers is exhausted.
4. When the legitimate leader runs the real co-sign round, every burned validator declines ("co-sign count limit reached"), `gather_signatures` never reaches threshold, and settlement fails after all retries. The counters are keyed by nullifier (not request id), never decrease, and have no TTL — re-proving with a fresh canonical id does not help. In a 2-of-2 validator set, burning the single co-signer fully freezes that spend.

**Impact:** An unprivileged network peer with no stake and no keys can indefinitely halt a targeted, already-quorum-approved settlement on every validator it can reach — a griefing/liveness halt of settlement (Medium), defeating the gate that was added to prevent exactly this class of abuse.

**Note:** the counts map is size-bounded with arbitrary eviction (#626), so the freeze lasts until node restart or until the entries happen to be evicted under map pressure; on a quiet devnet node neither occurs in practice, and the budget can be re-burned at negligible cost.

**Suggested fix:** Require proof of wallet ownership at Discovery registration (e.g. a signature over `domain || NodeId || timestamp` verified at registration — the vote-signing primitives can be reused), or admit a NodeId as a validator only after it has produced a valid vote signed by the claimed wallet.

Reporter: zc_whitehat, contact digdcc@gmail.com. Payout (USDC, Solana): 8B73vBdUqXofypTpgGgJc85WCvvTFwWDqMtrsXaxVqem

## Instructions for Kuro (Autonomous Solver):
1. Use skills: `systematic-debugging` and `code-review-and-quality`.
2. Inspect the repository structure, locate relevant files, and understand the bug/feature.
3. Apply the minimal, robust fix or implementation.
4. Run any existing test suites or linters in the workspace.
5. Ensure the fix adheres to the project's existing code style.
6. When done, output a summary of files modified and verification results.
