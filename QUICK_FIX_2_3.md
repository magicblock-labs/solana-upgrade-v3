# Quick Fix Reference: ProcessedTransaction 2.2 → 3.x

## Critical Changes

### 1. RollbackAccounts Field Name
```rust
// ❌ OLD (2.2)
if let RollbackAccounts::FeePayerOnly { fee_payer_account } = &fo.rollback_accounts

// ✅ NEW (3.x)
if let RollbackAccounts::FeePayerOnly { fee_payer } = &fo.rollback_accounts
```

### 2. RollbackAccounts Field Type
```rust
// ❌ OLD (2.2) - fee_payer_account: AccountSharedData
let accounts_to_persist = &[(fee_payer, fee_payer_account.clone())];

// ✅ NEW (3.x) - fee_payer: KeyedAccountSharedData = (Pubkey, AccountSharedData)
let accounts_to_persist = &[fee_payer.clone()];
// OR unpack it:
let (pubkey, account) = fee_payer;
let accounts_to_persist = &[(pubkey.clone(), account.clone())];
```

### 3. LoadedTransaction.accounts Type
```rust
// Both work (type alias compatible):
let accounts: &[(Pubkey, AccountSharedData)] = &executed.loaded_transaction.accounts;
// But in 3.x: Vec<KeyedAccountSharedData> (alias, not direct tuple)
```

### 4. All RollbackAccounts Variants Now Keyed
```rust
// SameNonceAndFeePayer
match &rollback_accounts {
    RollbackAccounts::SameNonceAndFeePayer { nonce } => {
        let (pubkey, account_data) = nonce;  // Now keyed in 3.x
    }
}

// SeparateNonceAndFeePayer  
match &rollback_accounts {
    RollbackAccounts::SeparateNonceAndFeePayer { nonce, fee_payer } => {
        let (nonce_key, nonce_account) = nonce;
        let (payer_key, payer_account) = fee_payer;
    }
}
```

## Files to Update

### magicblock-processor/src/executor/processing.rs

**Line 292-302** - RollbackAccounts pattern match:
```rust
// BEFORE
if let RollbackAccounts::FeePayerOnly { fee_payer_account } =
    &fo.rollback_accounts
{
    return self.insert_and_notify(
        &[(fee_payer, fee_payer_account.clone())],
        is_replay,
        false,
    );
}

// AFTER
if let RollbackAccounts::FeePayerOnly { fee_payer } =
    &fo.rollback_accounts
{
    // fee_payer is now (Pubkey, AccountSharedData)
    return self.insert_and_notify(
        &[fee_payer.clone()],
        is_replay,
        false,
    );
}
```

## Compile Check
```bash
cargo check -p magicblock-processor
```

## Test
```bash
cargo nextest run --package magicblock-processor
```

## Impact Summary

| Item | Severity | Status |
|------|----------|--------|
| ProcessedTransaction enum | Low | ✅ No change |
| ExecutedTransaction struct | Low | ✅ No change |
| RollbackAccounts field names | **HIGH** | ❌ Must update |
| RollbackAccounts types | **HIGH** | ❌ Must update |
| LoadedTransaction visibility | Medium | ⚠️ Check if used |

## Estimated Fix Time
- ~10-15 minutes for the critical changes
- Pattern match updates: 2-3 files
- Type adjustments: 1-2 functions

---

**Reference**: See `2_3-processed-transaction.md` for detailed analysis.
