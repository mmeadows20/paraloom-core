# Bounty Objective: [BOUNTY] Medium: ShieldedPool supply monotonically inflates and             commitment tree desynchronizes due to missing output commitments and          balance debit in settle_transact_via_cosign
Target: paraloom-labs/paraloom-core Issue #1136
URL: https://github.com/paraloom-labs/paraloom-core/issues/1136

## Issue Description
**Describe the bug**                                                     
    In `src/node/mod.rs:2539-2547` (`settle_transact_via_cosign`), when a    
  transact settlement confirms on-chain (`if settled`), the node only invokes
  `pool.record_spent(approved.request.nullifiers)`.                          
                                                                             
    It completely omits updating the local `ShieldedPool` state for the newly
  created output notes and external withdrawals:                             
    1. `approved.request.output_commitments` are never added to `self.       
  commitment_tree`.                                                          
    2. When a withdrawal occurs (`approved.request.ext_amount < 0`), `self.  
  supplies` is never debited for the external payout amount.                 
                                                                             
    Impact:                                                                  
    - The node's commitment tree permanently desynchronizes from the on-chain
  Merkle tree. Subsequent attempts by clients to fetch witness paths for     
  newly created output notes via `pool.path(&commitment)` fail with          
  `CommitmentNotFound`.                                                      
    - The off-chain tracked shielded supply (`pool.supply_of(...)` / `pool.  
  total_supply()`) monotonically diverges from the on-chain vault balance    
  upon every withdrawal, permanently violating accounting invariants and     
  breaking `verify_consistency()`.                                           
                                                                             
    **To Reproduce**                                                         
    Steps to reproduce the behavior:                                         
    1. Start a Paraloom node with an active `ShieldedPool`.                  
    2. Submit and co-sign an `ApprovedTransact` containing a valid withdrawal
  (`ext_amount = -1_000_000_000` lamports) and two output commitments `[c0,  
  c1]`.                                                                      
    3. Wait for `settle_transact_via_cosign` to complete successfully on-    
  chain (`settled == true`).                                                 
    4. Inspect `pool.commitment_count()` and `pool.                          
  supply_of(NATIVE_SOL_ASSET)`:                                              
       - Notice output commitments `c0` and `c1` are missing from the tree   
  (`pool.path(&c0)` returns `CommitmentNotFound`).                           
       - Notice `pool.supply_of(NATIVE_SOL_ASSET)` was never decremented by 1
  SOL, retaining the pre-transact supply despite funds leaving the on-chain  
  vault.                                                                     
                                                                             
    **Expected behavior**                                                    
    When `settled == true` in `settle_transact_via_cosign`:                  
    1. `pool` should record the newly minted `output_commitments` into `self.
  commitment_tree`.                                                          
    2. If `approved.request.ext_amount < 0`, `pool.supplies` must be         
  decremented by `approved.request.ext_amount.unsigned_abs()` for the        
  corresponding asset (SOL or SPL mint).                                     
                                                                             
    **Environment (please complete the following information):**             
     - OS: Ubuntu 22.04 LTS / macOS 15.0                                     
     - Rust Version: 1.85.0+                                                 
     - Paraloom Version: 0.6.0 (commit `9e8c8e6`)                            
                                                                             
    **Additional context**                                                   
    - Affected code: `src/node/mod.rs:2539-2547`                             
    - Bug Bounty Payout Wallet (Base USDC):                                  
  `0xbd42ca36676550e489d746607cb030422a153eb6`

## Instructions for Kuro (Autonomous Solver):
1. Use skills: `systematic-debugging` and `code-review-and-quality`.
2. Inspect the repository structure, locate relevant files, and understand the bug/feature.
3. Apply the minimal, robust fix or implementation.
4. Run any existing test suites or linters in the workspace.
5. Ensure the fix adheres to the project's existing code style.
6. When done, output a summary of files modified and verification results.
