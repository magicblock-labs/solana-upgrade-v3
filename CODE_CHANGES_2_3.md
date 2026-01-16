# Exact Code Changes: ProcessedTransaction 2.2 → 3.x

## File: magicblock-processor/src/executor/processing.rs

### Change 1: Line 292-302 (RollbackAccounts Pattern Match)

**Current Code (2.2)**:
```rust
ProcessedTransaction::FeesOnly(fo) => {
    if let RollbackAccounts::FeePayerOnly { fee_payer_account } =
        &fo.rollback_accounts
    {
        // Temporary slice construction to match expected type
        // This is slightly inefficient but safe; the vector here is tiny (1 item)
        return self.insert_and_notify(
            &[(fee_payer, fee_payer_account.clone())],
            is_replay,
            false,
        );
    }
    return Ok(());
}
```

**Updated Code (3.x)**:
```rust
ProcessedTransaction::FeesOnly(fo) => {
    if let RollbackAccounts::FeePayerOnly { fee_payer } =
        &fo.rollback_accounts
    {
        // In Solana 3.x, fee_payer is KeyedAccountSharedData = (Pubkey, AccountSharedData)
        // so we can pass it directly since it already includes the pubkey
        return self.insert_and_notify(
            &[fee_payer.clone()],
            is_replay,
            false,
        );
    }
    return Ok(());
}
```

**Changes**:
1. Field name: `fee_payer_account` → `fee_payer`
2. Type: Now `fee_payer: (Pubkey, AccountSharedData)` instead of just `AccountSharedData`
3. Argument: `&[(fee_payer, fee_payer_account.clone())]` → `&[fee_payer.clone()]`

### Change 2: Optional - Handle Other RollbackAccounts Variants

If the code is extended to handle `SameNonceAndFeePayer` or `SeparateNonceAndFeePayer`:

**Pattern (if needed)**:
```rust
ProcessedTransaction::FeesOnly(fo) => {
    match &fo.rollback_accounts {
        RollbackAccounts::FeePayerOnly { fee_payer } => {
            // fee_payer: (Pubkey, AccountSharedData)
            return self.insert_and_notify(
                &[fee_payer.clone()],
                is_replay,
                false,
            );
        }
        RollbackAccounts::SameNonceAndFeePayer { nonce } => {
            // nonce: (Pubkey, AccountSharedData)
            // Single account serves as both nonce and fee payer
            return self.insert_and_notify(
                &[nonce.clone()],
                is_replay,
                false,
            );
        }
        RollbackAccounts::SeparateNonceAndFeePayer { nonce, fee_payer } => {
            // Both are (Pubkey, AccountSharedData)
            // May need to update both accounts
            return self.insert_and_notify(
                &[nonce.clone(), fee_payer.clone()],
                is_replay,
                false,
            );
        }
    }
    return Ok(());
}
```

### Change 3: Verify insert_and_notify Signature (No Changes Needed)

The `insert_and_notify` function already accepts `&[(Pubkey, AccountSharedData)]`:

```rust
fn insert_and_notify(
    &self,
    accounts: &[(Pubkey, solana_account::AccountSharedData)],
    is_replay: bool,
    privileged: bool,
) -> AccountsDbResult<()>
```

This signature is **compatible** with:
- 2.2 format: Manually constructed tuples `&[(fee_payer, fee_payer_account.clone())]`
- 3.x format: `KeyedAccountSharedData` which is `(Pubkey, AccountSharedData)`

**No changes needed** to the function signature or other callers.

### Change 4: Verify LoadedTransaction Account Access (No Changes Needed)

The current code at lines 286-289:

```rust
if !succeeded {
    // Only charge fee payer on failure
    &executed.loaded_transaction.accounts[..1]
} else {
    &executed.loaded_transaction.accounts
}
```

This is **compatible** with both:
- 2.2: `Vec<(Pubkey, AccountSharedData)>`
- 3.x: `Vec<KeyedAccountSharedData>` (type alias for the same tuple)

**No changes needed** - tuple destructuring remains the same.

---

## Summary of Changes

| Location | Type | Old | New | Priority |
|----------|------|-----|-----|----------|
| Line 292 | Field name | `fee_payer_account` | `fee_payer` | **HIGH** |
| Line 298 | Argument | `&[(fee_payer, fee_payer_account.clone())]` | `&[fee_payer.clone()]` | **HIGH** |
| Line 348 (if used) | Type check | `AccountSharedData` | `KeyedAccountSharedData` | Low |

---

## Testing Checklist

After making changes:

1. **Syntax Check**:
   ```bash
   cargo check -p magicblock-processor
   ```

2. **Compilation**:
   ```bash
   cargo build -p magicblock-processor
   ```

3. **Unit Tests**:
   ```bash
   cargo nextest run -p magicblock-processor
   ```

4. **Integration Tests** (if applicable):
   ```bash
   RUN_TESTS='magicblock_api' make -C test-integration test
   ```

---

## Dependency Versions

Ensure you're using compatible versions:

**In Cargo.toml** (workspace dependencies):
```toml
[workspace.dependencies.solana-svm]
git = "https://github.com/magicblock-labs/magicblock-svm.git"
rev = "3e9456ec4"  # Current 3.x version
features = ["dev-context-only-utils"]
```

For Solana 3.x, the following should also be updated:
```toml
solana-transaction-context = { version = "3.x" }  # For KeyedAccountSharedData type
solana-account = { version = "3.x" }  # For AccountSharedData type
```

---

## Common Pitfalls

### Pitfall 1: Forgetting the Field Rename
```rust
// ❌ WRONG - Will not compile in 3.x
if let RollbackAccounts::FeePayerOnly { fee_payer_account } = ...

// ✅ CORRECT
if let RollbackAccounts::FeePayerOnly { fee_payer } = ...
```

### Pitfall 2: Double-Wrapping the Tuple
```rust
// ❌ WRONG - Creates ((Pubkey, AccountSharedData), ...)
&[(fee_payer.clone())]  // This is correct, not double-wrapped!

// ✅ CORRECT - Single keyed account
&[fee_payer.clone()]
```

### Pitfall 3: Accessing Fields That Are Now `pub(crate)`
```rust
// ❌ MAY FAIL in 3.x (outside SVM crate)
let indices = &executed.loaded_transaction.program_indices;
let budget = &executed.loaded_transaction.compute_budget;

// ✅ Use public methods instead (if available)
// Check SVM API for accessors
```

---

## Migration Timeline

1. **Update** `magicblock-processor/src/executor/processing.rs` (5-10 min)
2. **Compile** and fix any remaining type errors (5 min)
3. **Test** locally (5 min)
4. **Run** integration tests if needed (10-15 min)
5. **Verify** with MagicBlock validator (depends on your test setup)

**Total estimated time**: 30-45 minutes

---

## Rollback Plan

If anything breaks, the original 2.2 code pattern is preserved in version control. No permanent changes to the validator logic are required—only mechanical updates to field names and type handling.

```bash
git checkout <original-branch> -- magicblock-processor/src/executor/processing.rs
```

---

**Generated**: Solana 3.x ProcessedTransaction Migration
**For**: MagicBlock Validator
**Status**: Ready for implementation
