# Amsterdam EIP Implementation Notes

## Current Status (2026-01-30)
- **25 failing tests** (down from 57, was ~260+ initially)
- Most failures are BlockAccessListHashMismatch (EIP-7928 BAL)
- Some failures are ReceiptsRootMismatch (possibly EIP-7708 related)

### Recent Fixes

#### Fix 9: Self-Transfer TX Transfer Logs (29→28)
- **Issue**: Self-transfers (TX where origin == to) were not emitting transfer logs
- **Root Cause**: Code checked `from != to` before emitting, but EIP-7708 expects logs for all value transfers
- **Fix**: Remove the `from != to` check in `transfer_value` for TX-level transfers
- **File**: `crates/vm/levm/src/hooks/default_hook.rs`
- **Test**: `test_bal_self_transfer`

#### Fix 10: Balance Change Checkpoint/Restore (28→25)
- **Issue**: When a nested CALL reverts, balance changes were not being correctly restored
- **Root Cause**: `record_balance_change` updated the existing entry for the same block_access_index instead of pushing a new one. Since checkpoint only saves vector lengths (not values), restoring didn't fix the modified value.
- **Example flow**:
  1. TX transfers value → balance_changes = [(1, 2)]
  2. CREATE checkpoint (length = 1)
  3. CREATE transfers → balance_changes = [(1, 1)] (UPDATED, not pushed!)
  4. CREATE reverts, restore → still [(1, 1)] (wrong!)
- **Fix**: Always push new entries in `record_balance_change`, deduplication happens in `build()`
- **File**: `crates/common/types/block_access_list.rs`
- **Tests**: `test_failed_create_with_value_no_log`, `test_create_out_of_gas_no_log`, `test_failed_inner_operation_no_log`

#### Fix 1: Coinbase BAL Recording (57→50)
- **Issue**: Coinbase was being recorded as touched for withdrawal-only blocks
- **Fix**: Only record coinbase when there are actual transactions (coinbase receives TX fees, not withdrawal amounts)
- **File**: `crates/vm/backends/levm/mod.rs`

#### Fix 2: Empty Bytecode Code Changes (50→45)
- **Issue**: `update_account_bytecode` was recording code changes for empty bytecode (e.g., when CREATE deploys empty code)
- **Fix**: Only record code changes when bytecode is non-empty
- **File**: `crates/vm/levm/src/db/gen_db.rs`

#### Fix 3: CREATE Early Failure BAL Recording (45→43)
- **Issue**: CREATE was recording the new contract address in BAL before checking if CREATE can proceed (e.g., insufficient balance)
- **Fix**: Move `record_touched_address` call to AFTER early validation checks (OutOfFund, MaxDepth, MaxNonce)
- **File**: `crates/vm/levm/src/opcode_handlers/system.rs`
- **Test**: `test_bal_create_early_failure`

#### Fix 4: Deduplicate BAL Changes per Transaction (43→39)
- **Issue**: Multiple nonce/balance/code changes at the same block_access_index were all being recorded
- **Fix**: For each (address, index), only record the FINAL value, not intermediate changes
- **File**: `crates/common/types/block_access_list.rs`
- **Tests**: Multiple 7702 delegation tests

#### Fix 5: SSTORE Storage Read Timing (39→38)
- **Issue**: Storage read was only recorded after main gas check passed, but EIP-7928 says reads should be recorded after implicit SLOAD even if main gas check fails
- **Fix**: Record storage read immediately after `access_storage_slot` (implicit SLOAD), before main gas check
- **File**: `crates/vm/levm/src/opcode_handlers/stack_memory_storage_flow.rs`
- **Test**: `test_bal_sstore_and_oog[oog_at_exact_gas_minus_1]`

#### Fix 6: EIP-7702 Delegation Target BAL Recording (38→33)
- **Issue**: When accessing an EIP-7702 delegated account, only the delegating account was recorded in BAL, not the delegation target
- **Fix**: Record delegation target address as touched when `eip7702_get_code` finds a delegation
- **File**: `crates/vm/levm/src/utils.rs`
- **Tests**: Multiple 7702 delegation tests including `test_transfer_to_delegated_account_emits_log`

#### Fix 7: Top-Level Transaction Revert BAL Handling (33→32)
- **Issue**: When a top-level transaction reverts, BAL state changes were not being reverted (storage writes should become reads)
- **Fix**: Take BAL checkpoint before execution and restore on revert
- **File**: `crates/vm/levm/src/vm.rs`
- **Test**: `test_simple_gas_accounting[refund_tx_reverts]`

#### Fix 8: Top-Level Transaction Failure BAL Handling (32→29)
- **Issue**: Original checkpoint/restore approach was too aggressive - it restored ALL BAL state including valid storage reads. For SSTORE OOG cases, the implicit SLOAD happened (recording a read) but the checkpoint/restore wiped it out.
- **Root Cause**: Checkpoint was taken at start of tx (when reads were empty), so restore wiped all reads that happened during execution.
- **Fix**: Replace checkpoint/restore with `convert_writes_to_reads_for_tx_failure()`:
  - Only converts writes from the CURRENT transaction (preserves system contract writes at index 0)
  - Adds written slots to storage_reads (they were accessed)
  - Removes current tx's writes (they didn't persist)
  - Preserves existing storage reads (reads are always valid accesses)
- **Files**: `crates/common/types/block_access_list.rs`, `crates/vm/levm/src/vm.rs`
- **Tests**: `test_bal_sstore_and_oog`, `test_simple_gas_accounting`, EIP-7778 gas accounting tests

### Remaining Failures (25)

**EIP-7708 Transfer Logs (5 tests):**
- `test_call_opcodes_transfer_log_behavior` - CALLCODE variant fails with ReceiptsRootMismatch
- `test_inner_call_succeeds_outer_reverts_no_log` - Complex revert scenario
- `test_selfdestruct_during_initcode` - SELFDESTRUCT during CREATE
- `test_selfdestruct_to_system_address` - Special address handling
- `test_transfer_to_special_address` - Special address handling

**EIP-7928 BAL Tests (20 tests):**
- System contracts: test_bal_4788_query, test_bal_7002_* (4 tests), test_bal_consolidation_*, test_bal_withdrawal_*
- EIP-7702 + OOG: test_bal_*_7702_delegation_and_oog (4 tests)
- EIP-7702 delegation: test_bal_7702_delegation_clear, test_bal_7702_double_auth_reset
- CALLCODE: test_bal_callcode_nested_value_transfer, test_bal_callcode_no_delegation_and_oog_before_target_access
- Nested calls: test_bal_aborted_account_access, test_bal_call_no_delegation_oog_after_target_access, test_bal_nested_delegatecall_storage_writes_net_zero
- Other: test_bal_nonexistent_account_access_value_transfer, test_bal_create_selfdestruct_to_self_with_call

### Known Remaining Issues

#### Nested Call Checkpoint/Restore
- Tests: test_bal_aborted_*, test_bal_call_no_delegation_oog_after_target_access
- Issue: When nested CALLs revert, BAL changes may not be properly restored in some edge cases
- Root cause: Timing of checkpoint creation vs state changes

#### EIP-7702 Delegation + OOG
- Tests: test_bal_*_7702_delegation_and_oog (4 tests)
- Issue: Complex interaction between EIP-7702 delegation lookup and OOG handling
- May need special handling for delegation access that fails due to gas

#### CALLCODE-specific
- Tests: test_call_opcodes_transfer_log_behavior[CALLCODE], test_bal_callcode_nested_value_transfer
- Issue: CALLCODE has unique semantics where value is transferred to self (to == from)
- May need different BAL/log handling for CALLCODE vs CALL

## EIP-7708: ETH Transfers Emit a Log

### Key Findings

1. **Transfer log emitters** (per EF tests, which differ slightly from EIP text):
   - Transaction value transfers (origin → to) for non-create txs
   - Contract creation transactions (origin → new_contract)
   - CALL opcode with value (caller → callee)
   - CREATE/CREATE2 opcodes with value (deployer → new_address) - EIP text doesn't mention this explicitly, but EF tests expect it
   - SELFDESTRUCT to different address (contract → beneficiary)
   - SELFDESTRUCT to self with balance (emits Selfdestruct log, not Transfer log)

2. **What does NOT emit logs**:
   - CALLCODE - executes in caller's context, `to == from`, no actual value movement
   - DELEGATECALL - no value transfer capability
   - STATICCALL - no value transfer capability

3. **Selfdestruct to self behavior**:
   - Per EIP-7708: Selfdestruct log is emitted when SELFDESTRUCT to self is triggered with non-zero balance
   - This happens REGARDLESS of whether contract was created in same transaction (EIP-6780)
   - The `is_account_created` check was incorrectly limiting this

4. **Log format**:
   - Address: `0xfffffffffffffffffffffffffffffffffffffffe` (EIP7708_SYSTEM_ADDRESS)
   - Transfer: LOG3 with topics [keccak256('Transfer(address,address,uint256)'), from, to], data=value
   - Selfdestruct: LOG2 with topics [keccak256('Selfdestruct(address,uint256)'), contract], data=balance

### Remaining EIP-7708 Issues

#### CALLCODE Variant in test_call_opcodes_transfer_log_behavior (ReceiptsRootMismatch)
- **Observation**: CALL, DELEGATECALL, STATICCALL variants pass; only CALLCODE fails
- **Investigation**:
  - Transfer log content is correct (from/to addresses, value, topic hash all match expected bloom)
  - Bloom filter computation matches expected
  - Gas values match (cumulative_gas_used)
  - Receipt encoding includes gas_spent for Amsterdam+
- **Possible issues to investigate**:
  - gas_spent field encoding might be formatted differently
  - Receipt RLP structure might need adjustment for Amsterdam
  - CALLCODE-specific value transfer semantics
- The CALLCODE opcode correctly does NOT emit a transfer log (from==to), but the TX-level transfer does

### Fixed Issues
- `test_selfdestruct_to_self_emits_log` - Fixed by removing `is_account_created` check for selfdestruct to self
- `test_selfdestruct_log_at_fork_transition` - Fixed along with selfdestruct changes

## EIP-7778: Block Gas Accounting without Refunds

### Key Findings

1. **Two gas fields in receipts**:
   - `cumulative_gas_used`: Pre-refund gas (for block-level accounting)
   - `gas_spent`: Post-refund gas (what user actually pays) - new field for Amsterdam+

2. **EIP-7623 gas floor**:
   - Minimum gas based on calldata costs
   - Must be applied to BOTH `gas_used` AND `gas_spent` for Amsterdam+
   - Previously only applied to `gas_spent`

3. **Receipt encoding**:
   - `gas_spent` is included in RLP encoding when present (Amsterdam+)
   - Format: [succeeded, cumulative_gas_used, bloom, logs, gas_spent]

## EIP-7928: Block-Level Access Lists (BAL)

### Key Findings
- Most failures (52+) are BlockAccessListHashMismatch
- BAL tracks all state accesses per block
- Hash computation needs investigation

### BAL Structure (from `block_access_list.rs`)
- `BlockAccessList` contains accounts with their changes
- Each account tracks: storage_changes, balance_changes, nonce_changes, code_changes
- Changes are indexed by `block_access_index` (u16): 0=pre-exec, 1..n=tx indices, n+1=post-exec
- Accounts are RLP-encoded in sorted order by address
- Hash is keccak256 of the RLP-encoded list

### Potential Issues to Investigate
1. Missing state accesses being recorded
2. Wrong ordering in RLP encoding
3. Wrong block_access_index values
4. Missing or extra accounts/changes in BAL
5. SYSTEM_ADDRESS handling (excluded unless it has actual state changes)

## Files Modified

- `crates/vm/levm/src/hooks/default_hook.rs` - Transaction-level transfer logs, gas accounting
- `crates/vm/levm/src/opcode_handlers/system.rs` - CREATE, CALL, SELFDESTRUCT log emission
- `crates/vm/levm/src/execution_handlers.rs` - Contract creation transaction log
- `crates/common/types/receipt.rs` - Receipt encoding with gas_spent field
- `crates/common/validation.rs` - Block validation

## Quirks and Edge Cases

1. **CREATE opcodes emit logs** even though EIP-7708 text doesn't explicitly mention them
2. **Selfdestruct to self** emits log even when contract isn't destroyed (per EIP-6780)
3. **CALLCODE** uses same gas as CALL but doesn't emit transfer log (to == from)

## Remaining Failures Investigation (29 tests)

### EIP-7708 Transfer Logs (7 tests)
- `test_failed_create_with_value_no_log` - CREATE fails (initcode reverts/invalid), no transfer log expected
- `test_create_out_of_gas_no_log` - CREATE OOGs, no transfer log
- `test_failed_inner_operation_no_log` - Inner operation fails
- `test_inner_call_succeeds_outer_reverts_no_log` - Complex revert scenario
- `test_selfdestruct_during_initcode` - SELFDESTRUCT during CREATE
- `test_transfer_to_special_address` - Special address handling
- `test_call_opcodes_transfer_log_behavior` - CALL opcode variants

### EIP-7002 System Contract (4 tests)
- `test_bal_7002_clean_sweep`, `partial_sweep`, `request_from_contract`, `request_invalid`
- Issue: System contract (withdrawal request predeploy) BAL recording
- The system contract shows balance_changes at idx=0 with bal=1 - needs investigation
- Request extraction happens at block_access_index 0 (pre-execution)

### EIP-7702 Delegation Tests (6 tests)
- `test_bal_7702_delegation_clear`, `test_bal_7702_double_auth_reset`
- `test_bal_*_7702_delegation_and_oog` (4 variants: call/callcode/delegatecall/staticcall)
- Issue: Complex interaction between EIP-7702 delegation and OOG handling
- Delegation target recording may need adjustment

### Nested Call/Revert (3 tests)
- `test_bal_aborted_account_access`
- `test_bal_call_no_delegation_oog_after_target_access`
- `test_bal_nested_delegatecall_storage_writes_net_zero`
- Issue: Checkpoint/restore timing with nested calls

### CALLCODE Specific (2 tests)
- `test_bal_callcode_nested_value_transfer`
- `test_bal_callcode_no_delegation_and_oog_before_target_access`
- Issue: CALLCODE has unique semantics (to == from for value transfer)

### Other BAL Edge Cases (7 tests)
- `test_bal_4788_query` - Beacon root system contract
- `test_bal_consolidation_contract_cross_index` - EIP-7251 consolidation
- `test_bal_self_transfer` - ReceiptsRootMismatch (not BAL issue)
- `test_bal_nonexistent_account_access_value_transfer`
- `test_bal_create_selfdestruct_to_self_with_call`

### Key Observations

1. **System contract request extraction** happens at block_access_index 0, after transactions but as part of "pre-execution" phase per EIP-7928.

2. **CREATE failures** still show touched addresses in BAL even when initcode reverts - this may or may not be correct per EIP-7928.

3. **EIP-7702 + OOG** tests involve complex gas accounting where delegation lookup itself can cause OOG.

4. **ReceiptsRootMismatch** vs **BlockAccessListHashMismatch** - some tests fail on receipt encoding, not BAL structure.
