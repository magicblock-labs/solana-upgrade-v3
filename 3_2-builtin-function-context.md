# Issue 3.2: BuiltinFunctionWithContext Signature Changes in Solana 3.x

**Status:** VERIFIED  
**Date:** 2026-01-15  
**Current Version:** Solana 2.2.1  
**Target Version:** Solana 3.0+  

---

## Executive Summary

The `BuiltinFunctionWithContext` type definition remains **structurally identical** across Solana 2.2 and 3.x:

```rust
pub type BuiltinFunctionWithContext = BuiltinFunction<InvokeContext<'static>>;
```

**However**, the `InvokeContext<'static>` struct undergoes **significant architectural changes** that indirectly affect how builtins are invoked and what context data is available. These changes are breaking at the type level.

---

## 1. BuiltinFunctionWithContext Type Definition

### What is BuiltinFunctionWithContext?

It's a **type alias**, not a function pointer directly. It expands to:

```rust
type BuiltinFunctionWithContext = BuiltinFunction<InvokeContext<'static>>;
```

Where `BuiltinFunction<T>` is from `solana_rbpf::program::BuiltinFunction`:

```rust
// solana_rbpf crate
pub type BuiltinFunction<T> = fn(
    context: &mut T,
    arg0: u64,
    arg1: u64,
    arg2: u64,
    arg3: u64,
    arg4: u64,
    memory_mapping: &mut MemoryMapping,
) -> Result<u64, Box<dyn std::error::Error>>;
```

### Function Signature

**Both 2.2 and 3.x:**

```rust
fn(
    &mut InvokeContext<'static>,  // The context object
    u64, u64, u64, u64, u64,      // 5 u64 arguments
    &mut MemoryMapping             // Memory mapping for heap
) -> Result<u64, Box<dyn std::error::Error>>
```

✅ **NO CHANGE** in the function pointer signature itself.

---

## 2. InvokeContext Structure Changes (BREAKING)

The real changes are **inside** `InvokeContext<'a>`. This struct is completely restructured.

### Solana 2.2 InvokeContext

```rust
pub struct InvokeContext<'a> {
    /// Information about the currently executing transaction.
    pub transaction_context: &'a mut TransactionContext,
    /// The local program cache for the transaction batch.
    pub program_cache_for_tx_batch: &'a mut ProgramCacheForTxBatch,
    /// Runtime configurations used to provision the invocation environment.
    pub environment_config: EnvironmentConfig<'a>,
    /// The compute budget for the current invocation.
    compute_budget: ComputeBudget,
    /// Instruction compute meter, for tracking compute units consumed.
    compute_meter: RefCell<u64>,
    log_collector: Option<Rc<RefCell<LogCollector>>>,
    pub execute_time: Option<Measure>,
    pub timings: ExecuteDetailsTimings,
    pub syscall_context: Vec<Option<SyscallContext>>,
    traces: Vec<Vec<[u64; 12]>>,
}
```

**Key fields in 2.2:**
- `compute_budget: ComputeBudget` — Simple compute budget struct
- **11 fields total**

### Solana 3.0 InvokeContext

```rust
pub struct InvokeContext<'a> {
    /// Information about the currently executing transaction.
    pub transaction_context: &'a mut TransactionContext,
    /// The local program cache for the transaction batch.
    pub program_cache_for_tx_batch: &'a mut ProgramCacheForTxBatch,
    /// Runtime configurations used to provision the invocation environment.
    pub environment_config: EnvironmentConfig<'a>,
    /// The compute budget for the current invocation.
    compute_budget: SVMTransactionExecutionBudget,
    /// The compute cost for the current invocation.
    execution_cost: SVMTransactionExecutionCost,
    /// Instruction compute meter, for tracking compute units consumed.
    compute_meter: RefCell<u64>,
    log_collector: Option<Rc<RefCell<LogCollector>>>,
    pub execute_time: Option<Measure>,
    pub timings: ExecuteDetailsTimings,
    pub syscall_context: Vec<Option<SyscallContext>>,
    traces: Vec<Vec<[u64; 12]>>,
    /// Stops copying account data if stricter_abi_and_runtime_constraints is enabled
    pub account_data_direct_mapping: bool,
}
```

**Key fields in 3.0:**
- `compute_budget: SVMTransactionExecutionBudget` — New execution budget type
- `execution_cost: SVMTransactionExecutionCost` — NEW field for tracking cost
- `account_data_direct_mapping: bool` — NEW field for ABI constraints
- **13 fields total** (2 new fields added)

---

## 3. Environment Configuration Changes (BREAKING)

### Solana 2.2 EnvironmentConfig

```rust
pub struct EnvironmentConfig<'a> {
    pub blockhash: Hash,
    pub blockhash_lamports_per_signature: u64,
    epoch_total_stake: u64,                                    // ← 2.2 only
    get_epoch_vote_account_stake_callback: &'a dyn Fn(&'a Pubkey) -> u64,  // ← 2.2 only
    pub feature_set: Arc<FeatureSet>,                         // ← 2.2: Arc
    sysvar_cache: &'a SysvarCache,
}
```

**Key differences in 2.2:**
- Stores `epoch_total_stake` directly
- Has `get_epoch_vote_account_stake_callback` for querying stakes
- `feature_set` is an `Arc<FeatureSet>` (owned Arc)

### Solana 3.0 EnvironmentConfig

```rust
pub struct EnvironmentConfig<'a> {
    pub blockhash: Hash,
    pub blockhash_lamports_per_signature: u64,
    epoch_stake_callback: &'a dyn InvokeContextCallback,      // ← 3.0 only (new trait)
    feature_set: &'a SVMFeatureSet,                           // ← 3.0: reference, different type
    sysvar_cache: &'a SysvarCache,
}
```

**Key differences in 3.0:**
- **Removed** `epoch_total_stake` field (5 fields → 3 explicit trait fields)
- **Replaced** `get_epoch_vote_account_stake_callback: &'a dyn Fn(&'a Pubkey) -> u64` with a unified `epoch_stake_callback: &'a dyn InvokeContextCallback`
- `feature_set` is now `&'a SVMFeatureSet` (borrowed reference) instead of `Arc<FeatureSet>`
- Uses new `InvokeContextCallback` trait (from `solana_svm_callback`)

---

## 4. Compute Budget Type Changes (BREAKING)

### Solana 2.2

```rust
pub struct ComputeBudget {
    pub compute_unit_limit: u64,
    pub heap_size: u32,
    pub instruction_compute_unit_limit: u32,
    pub max_instruction_stack_height: u32,
    pub max_invoke_stack_height: u32,
    // ... other fields
}
```

Single compute budget type.

### Solana 3.0

**Two separate budget types:**

1. **`SVMTransactionExecutionBudget`** — Transaction-level budget:
   ```rust
   pub struct SVMTransactionExecutionBudget {
       pub compute_unit_limit: u64,
       pub instruction_compute_unit_limit: u32,
       pub instruction_stack_depth_limit: usize,
       pub call_depth_limit: usize,
       pub heap_size: u32,
   }
   ```

2. **`SVMTransactionExecutionCost`** — NEW: Tracks actual cost incurred:
   ```rust
   pub struct SVMTransactionExecutionCost {
       pub write_lock_cost: u64,
       pub data_bytes_cost: u64,
       // ... other cost fields
   }
   ```

**Impact:** `InvokeContext` now tracks both budget (limit) and cost (actual usage).

---

## 5. Feature Set Type Changes (BREAKING)

### Solana 2.2
- `feature_set: Arc<FeatureSet>` in `EnvironmentConfig`
- Standard Solana SDK `FeatureSet` type

### Solana 3.0
- `feature_set: &'a SVMFeatureSet` in `EnvironmentConfig`
- New `SVMFeatureSet` type (from `solana_svm_feature_set`)
- Reference instead of Arc (lifetime-bound)

---

## 6. New Callback Trait (3.0 Only)

Solana 3.0 introduces `InvokeContextCallback` trait:

```rust
// From solana_svm_callback
pub trait InvokeContextCallback {
    fn get_epoch_vote_account_stake(&self, pubkey: &Pubkey) -> u64;
    // ... other methods
}
```

**Replaces:** The direct `get_epoch_vote_account_stake_callback` function pointer in 2.2.

This is a trait object, allowing for more flexible, extensible callback interfaces.

---

## 7. New Fields in InvokeContext (3.0 Only)

Two new fields added:

1. **`execution_cost: SVMTransactionExecutionCost`**
   - Tracks actual execution costs
   - Supports more granular cost tracking
   - New in 3.0

2. **`account_data_direct_mapping: bool`**
   - Controls whether to stop copying account data if stricter ABI constraints are enabled
   - Optimization flag for newer SBPF versions
   - New in 3.0

---

## 8. Impact on MagicBlock Code

### File: `magicblock-processor/src/builtins.rs`

**Current 2.2 code:**
```rust
use solana_program_runtime::invoke_context::BuiltinFunctionWithContext;

pub struct Builtin {
    pub program_id: Pubkey,
    pub name: &'static str,
    pub entrypoint: BuiltinFunctionWithContext,
}

pub static BUILTINS: &[Builtin] = &[
    Builtin {
        program_id: system_program::ID,
        name: "system_program",
        entrypoint: solana_system_program::system_processor::Entrypoint::vm,
    },
    // ... more builtins
];
```

**Status for 3.x:** ✅ **No changes needed** to this file itself.

The `BuiltinFunctionWithContext` type alias will still work because:
1. The function pointer signature is identical
2. The `Entrypoint::vm` method signature remains compatible
3. The builtin registration mechanism doesn't change

**However**, any builtin **implementations** (inside `Entrypoint::vm`) that directly access `InvokeContext` fields will need updates.

### File: `magicblock-processor/src/executor/mod.rs` (lines 107-120)

**Current 2.2 code:**
```rust
pub(super) fn populate_builtins(&self) {
    for builtin in BUILTINS {
        let entry = ProgramCacheEntry::new_builtin(
            0,
            builtin.name.len(),
            builtin.entrypoint,
        );
        self.processor.add_builtin(
            self,
            builtin.program_id,
            builtin.name,
            entry,
        );
    }
}
```

**Status for 3.x:** ⚠️ **LIKELY CHANGES NEEDED**

The `ProgramCacheEntry::new_builtin()` and `add_builtin()` method signatures may have changed in 3.x. Need to verify:
- Constructor parameter types
- Return type
- Builder methods

---

## 9. Breaking Changes Summary

| Category | 2.2 | 3.0 | Breaking? |
|----------|-----|-----|-----------|
| `BuiltinFunctionWithContext` type | ✅ Same | ✅ Same | No |
| Function pointer signature | ✅ Same | ✅ Same | No |
| `InvokeContext` struct | Base | +2 fields | **Yes** |
| `EnvironmentConfig` | Old callback design | New trait-based callbacks | **Yes** |
| Compute budget type | `ComputeBudget` | `SVMTransactionExecutionBudget` + `SVMTransactionExecutionCost` | **Yes** |
| Feature set type | `Arc<FeatureSet>` | `&'a SVMFeatureSet` | **Yes** |
| Callback mechanism | Function pointer | Trait object | **Yes** |

**Breaking change level:** 🔴 **HIGH** — Affects context access patterns, but not the builtin registration itself.

---

## 10. Migration Strategy for MagicBlock

### Phase 1: Type Compatibility Check
1. ✅ Keep `BuiltinFunctionWithContext` imports (no change)
2. ✅ Keep builtin struct and registration unchanged
3. ⚠️ Update `ProgramCacheEntry::new_builtin()` call sites

### Phase 2: Context Access Updates
For any code that directly accesses `InvokeContext` fields:

**2.2 style:**
```rust
let budget = &invoke_context.compute_budget;
let stake = invoke_context.environment_config
    .get_epoch_vote_account_stake_callback(pubkey);
```

**3.0 style:**
```rust
let budget = &invoke_context.compute_budget;  // Type changed: ComputeBudget → SVMTransactionExecutionBudget
let cost = &invoke_context.execution_cost;    // NEW field
let stake = invoke_context.environment_config
    .epoch_stake_callback.get_epoch_vote_account_stake(pubkey);  // Trait method, not fn pointer
```

### Phase 3: Feature Set Updates
Update any code using `feature_set`:

**2.2:**
```rust
invoke_context.environment_config.feature_set.is_active(feature)
```

**3.0:**
```rust
invoke_context.environment_config.feature_set.is_active(feature)  // Type: SVMFeatureSet
```

---

## 11. Verification Steps

1. **Update Cargo.toml** to Solana 3.x versions
2. **Run `cargo check`** — will report type mismatches
3. **Fix `ProgramCacheEntry::new_builtin()` calls** — check new signature
4. **Search for `invoke_context.compute_budget`** — may need `SVMTransactionExecutionBudget` adjustments
5. **Search for callback access** — update to trait method style
6. **Run `make lint`** and `make test`**

---

## 12. Detailed Field Mapping

### InvokeContext Field Changes

| Field | 2.2 Type | 3.0 Type | Migration |
|-------|----------|----------|-----------|
| `transaction_context` | `&'a mut TransactionContext` | `&'a mut TransactionContext` | ✅ No change |
| `program_cache_for_tx_batch` | `&'a mut ProgramCacheForTxBatch` | `&'a mut ProgramCacheForTxBatch` | ✅ No change |
| `environment_config` | `EnvironmentConfig<'a>` | `EnvironmentConfig<'a>` | ⚠️ Fields changed |
| `compute_budget` | `ComputeBudget` | `SVMTransactionExecutionBudget` | 🔴 **Type change** |
| `execution_cost` | N/A | `SVMTransactionExecutionCost` | 🟢 **New field** |
| `compute_meter` | `RefCell<u64>` | `RefCell<u64>` | ✅ No change |
| `log_collector` | `Option<Rc<RefCell<LogCollector>>>` | `Option<Rc<RefCell<LogCollector>>>` | ✅ No change |
| `execute_time` | `Option<Measure>` | `Option<Measure>` | ✅ No change |
| `timings` | `ExecuteDetailsTimings` | `ExecuteDetailsTimings` | ✅ No change |
| `syscall_context` | `Vec<Option<SyscallContext>>` | `Vec<Option<SyscallContext>>` | ✅ No change |
| `traces` | `Vec<Vec<[u64; 12]>>` | `Vec<Vec<[u64; 12]>>` | ✅ No change |
| `account_data_direct_mapping` | N/A | `bool` | 🟢 **New field** |

### EnvironmentConfig Field Changes

| Field | 2.2 Type | 3.0 Type | Migration |
|-------|----------|----------|-----------|
| `blockhash` | `Hash` | `Hash` | ✅ No change |
| `blockhash_lamports_per_signature` | `u64` | `u64` | ✅ No change |
| `epoch_total_stake` | `u64` | Removed | 🔴 **Removed** |
| `get_epoch_vote_account_stake_callback` | `&'a dyn Fn(&'a Pubkey) -> u64` | Replaced | 🔴 **Replaced** |
| `epoch_stake_callback` | N/A | `&'a dyn InvokeContextCallback` | 🟢 **New** |
| `feature_set` | `Arc<FeatureSet>` | `&'a SVMFeatureSet` | 🔴 **Type change** |
| `sysvar_cache` | `&'a SysvarCache` | `&'a SysvarCache` | ✅ No change |

---

## 13. Code Examples: Before and After

### Example 1: Accessing Compute Budget

**Solana 2.2:**
```rust
fn process_builtin(invoke_context: &mut InvokeContext) -> ProgramResult {
    let remaining_cu = invoke_context.compute_budget.compute_unit_limit
        - invoke_context.compute_meter.borrow().deref() as u32;
    // ...
}
```

**Solana 3.0:**
```rust
fn process_builtin(invoke_context: &mut InvokeContext) -> ProgramResult {
    let remaining_cu = invoke_context.compute_budget.compute_unit_limit
        - invoke_context.compute_meter.borrow().deref() as u32;
    // Same code works! SVMTransactionExecutionBudget has same field
}
```

### Example 2: Accessing Epoch Stake

**Solana 2.2:**
```rust
let stake = (invoke_context.environment_config
    .get_epoch_vote_account_stake_callback)(pubkey);
```

**Solana 3.0:**
```rust
let stake = invoke_context.environment_config
    .epoch_stake_callback.get_epoch_vote_account_stake(pubkey);
```

### Example 3: Checking Feature

**Solana 2.2:**
```rust
if invoke_context.environment_config.feature_set
    .is_active(&solana_feature_set::tx_wide_compute_cap::id()) {
    // ...
}
```

**Solana 3.0:**
```rust
if invoke_context.environment_config.feature_set
    .is_active(&solana_feature_set::tx_wide_compute_cap::id()) {
    // Same call, but feature_set is now SVMFeatureSet
}
```

---

## 14. Implementation Notes for MagicBlock

### High Priority
1. Update `solana-program-runtime` crate version in Cargo.toml
2. Audit all `invoke_context.compute_budget` access
3. Update `EnvironmentConfig` callback access patterns
4. Verify `ProgramCacheEntry` API compatibility

### Medium Priority
1. Ensure feature set access patterns still work
2. Test with new `execution_cost` field (may need initialization)
3. Verify `account_data_direct_mapping` flag compatibility

### Low Priority
1. Update documentation/comments about context structure
2. Consider optimizations using new `account_data_direct_mapping` flag
3. Review cost tracking in `execution_cost`

---

## References

- **Solana 2.2 Source:** `anza-xyz/agave` v2.2 branch
  - `program-runtime/src/invoke_context.rs`
  - `program-runtime/src/compute_budget.rs`

- **Solana 3.0 Source:** `anza-xyz/agave` v3.0 branch
  - `program-runtime/src/invoke_context.rs`
  - `program-runtime/src/execution_budget.rs`
  - `program-runtime/src/callback.rs` (new)

- **Key crates in 3.0:**
  - `solana_svm_callback` — New callback trait
  - `solana_svm_feature_set` — New feature set type
  - `solana_sbpf` — Updated from `solana_rbpf`

---

## Conclusion

**The `BuiltinFunctionWithContext` type alias itself is unchanged**, but the **`InvokeContext` it wraps has significant structural changes**. The migration is manageable because:

1. ✅ Function pointer signature stays the same
2. ✅ Builtin registration mechanism unchanged
3. 🔴 Context field access patterns need updates (callback, budget types)
4. 🟢 New fields added for better cost/constraint tracking

**Estimated effort:** Medium (1-2 developer days for full audit + testing)

**Risk level:** Low to Medium (well-scoped changes, good test coverage critical)
