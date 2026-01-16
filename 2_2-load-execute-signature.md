# Research Issue 2.2: load_and_execute_sanitized_transactions Signature Changes

## Summary

The `load_and_execute_sanitized_transactions` method signature **REMAINS THE SAME** between Solana 2.x and Solana 3.x (Agave 4.0), but the **parameter types changed dramatically**, particularly `CheckedTransactionDetails`.

---

## Current Code (MagicBlock 2.x - Solana 2.2.1)

### Signature
```rust
pub fn load_and_execute_sanitized_transactions<CB: TransactionProcessingCallback>(
    &self,
    callbacks: &CB,
    sanitized_txs: &[impl SVMTransaction],
    check_results: Vec<TransactionCheckResult>,
    environment: &TransactionProcessingEnvironment,
    config: &TransactionProcessingConfig,
) -> LoadAndExecuteSanitizedTransactionsOutput
```

### Current Usage (executor/processing.rs:131-137)
```rust
let checked = CheckedTransactionDetails::new(
    None,
    self.environment.fee_lamports_per_signature,
);
let mut output = self.processor.load_and_execute_sanitized_transactions(
    self,
    txn,
    vec![Ok(checked); 1],
    &self.environment,
    &self.config,
);
```

### CheckedTransactionDetails (2.2.1)
```rust
pub struct CheckedTransactionDetails {
    pub(crate) nonce: Option<NonceInfo>,
    pub(crate) lamports_per_signature: u64,
}

impl CheckedTransactionDetails {
    pub fn new(nonce: Option<NonceInfo>, lamports_per_signature: u64) -> Self {
        Self {
            nonce,
            lamports_per_signature,
        }
    }
}
```

### Type Aliases
```rust
pub type TransactionCheckResult = Result<CheckedTransactionDetails>;
```

---

## Target (Solana 3.x / Agave 4.0)

### Signature
```rust
pub fn load_and_execute_sanitized_transactions<CB: TransactionProcessingCallback>(
    &self,
    callbacks: &CB,
    sanitized_txs: &[impl SVMTransaction],
    check_results: Vec<TransactionCheckResult>,
    environment: &TransactionProcessingEnvironment,
    config: &TransactionProcessingConfig,
) -> LoadAndExecuteSanitizedTransactionsOutput
```

**✓ IDENTICAL** - No changes to function signature

### CheckedTransactionDetails (4.0-alpha.0)
```rust
pub struct CheckedTransactionDetails {
    pub(crate) nonce_address: Option<Pubkey>,
    pub(crate) compute_budget_and_limits: SVMTransactionExecutionAndFeeBudgetLimits,
}

impl CheckedTransactionDetails {
    pub fn new(
        nonce_address: Option<Pubkey>,
        compute_budget_and_limits: SVMTransactionExecutionAndFeeBudgetLimits,
    ) -> Self {
        Self {
            nonce_address,
            compute_budget_and_limits,
        }
    }
}

#[cfg(feature = "dev-context-only-utils")]
impl Default for CheckedTransactionDetails {
    fn default() -> Self {
        Self {
            nonce_address: None,
            compute_budget_and_limits: SVMTransactionExecutionAndFeeBudgetLimits {
                budget: SVMTransactionExecutionBudget::default(),
                loaded_accounts_data_size_limit: NonZeroU32::new(32)
                    .expect("Failed to set loaded_accounts_bytes"),
                fee_details: FeeDetails::default(),
            },
        }
    }
}
```

### New Type Structures Required
```rust
// Need to import/construct:
pub struct SVMTransactionExecutionAndFeeBudgetLimits {
    pub budget: SVMTransactionExecutionBudget,
    pub loaded_accounts_data_size_limit: NonZeroU32,
    pub fee_details: FeeDetails,
}

pub struct SVMTransactionExecutionBudget {
    // Contains compute_unit budget, heap size, etc.
}

pub struct FeeDetails {
    // Contains fee calculation details
}
```

---

## Parameter Type Changes

| Aspect | 2.2.1 (Current) | 4.0-alpha.0 (Target) | Impact |
|--------|---|---|---|
| **CheckedTransactionDetails::new() param 1** | `nonce: Option<NonceInfo>` | `nonce_address: Option<Pubkey>` | Parameter name AND type changed |
| **CheckedTransactionDetails::new() param 2** | `lamports_per_signature: u64` | `compute_budget_and_limits: SVMTransactionExecutionAndFeeBudgetLimits` | **BREAKING** - structure instead of scalar |
| **Function signature** | No change | No change | ✓ Compatible |
| **Return type** | `LoadAndExecuteSanitizedTransactionsOutput` | `LoadAndExecuteSanitizedTransactionsOutput` | ✓ Compatible |

---

## Migration Path

### Step 1: Update CheckedTransactionDetails Construction

**Before (2.2.1):**
```rust
let checked = CheckedTransactionDetails::new(
    None,
    self.environment.fee_lamports_per_signature,
);
```

**After (3.x/4.0):**
```rust
// Need to build SVMTransactionExecutionAndFeeBudgetLimits
let budget = SVMTransactionExecutionBudget {
    // Fill in budget details from environment
};

let fee_details = FeeDetails {
    // Fill in fee calculation details
};

let limits = SVMTransactionExecutionAndFeeBudgetLimits {
    budget,
    loaded_accounts_data_size_limit: NonZeroU32::new(32)
        .expect("Failed to set loaded_accounts_data_size_limit"),
    fee_details,
};

let checked = CheckedTransactionDetails::new(
    None,  // nonce_address
    limits, // compute_budget_and_limits
);
```

### Step 2: Update Environment Integration

Investigate which fields from `self.environment` map to:
- `SVMTransactionExecutionBudget` (compute units, heap)
- `FeeDetails` (lamports_per_signature mapping)

### Step 3: Verify Fee Handling

The `lamports_per_signature` is now embedded in `FeeDetails` within the budget structure, not a separate parameter. Ensure fee validation still works.

---

## Trait Bounds - No Changes

Both versions require:
```rust
CB: TransactionProcessingCallback
```

The callback implementation (`self` in the current code) maintains the same trait bounds. No changes to the `AccountLoader` trait required.

---

## Return Type Structure

```rust
pub struct LoadAndExecuteSanitizedTransactionsOutput {
    pub processing_results: Vec<TransactionProcessingResult>,
    pub balances: AccountsBalances,
}
```

✓ **No changes** between versions.

---

## Key Differences Summary

| Category | Status | Details |
|----------|--------|---------|
| **Function Signature** | ✓ Unchanged | Method parameters and order identical |
| **Generic Bounds** | ✓ Unchanged | `CB: TransactionProcessingCallback` |
| **Return Type** | ✓ Unchanged | `LoadAndExecuteSanitizedTransactionsOutput` |
| **CheckedTransactionDetails Fields** | ✗ BREAKING | `(Option<NonceInfo>, u64)` → `(Option<Pubkey>, SVMTransactionExecutionAndFeeBudgetLimits)` |
| **Input Parameter Types** | ✓ Mostly Unchanged | Only CheckedTransactionDetails impacts the call site |
| **Callback Trait** | ✓ Unchanged | AccountLoader implementation compatible |

---

## Critical Migration Notes

1. **No signature change** means the method call site looks the same
2. **Constructor parameters COMPLETELY CHANGED** - must build SVMTransactionExecutionAndFeeBudgetLimits
3. **Nonce handling changed**: was `Option<NonceInfo>`, now `Option<Pubkey>`
4. **Fee handling integrated**: lamports_per_signature now part of FeeDetails inside SVMTransactionExecutionAndFeeBudgetLimits
5. **New budget complexity**: Must populate SVMTransactionExecutionBudget fields from environment

---

## Files Affected

- **MagicBlock**: `magicblock-processor/src/executor/processing.rs:131-137` (process method)
- **Need to inspect**: 
  - How `self.environment` provides budget information
  - Whether compute unit limits are currently set elsewhere
  - How `self.environment.fee_lamports_per_signature` maps to new structure

---

## Next Steps

1. **Find Solana 3.x documentation** on `SVMTransactionExecutionBudget` and `FeeDetails` construction
2. **Trace environment/config** to see what fields contain budget info
3. **Build test migration** with sample values
4. **Validate that existing transaction validation still works** with new budget structure
