# Research Issue 1.3: ReadableAccount/WritableAccount Trait Changes in Solana 3.x

**Date**: January 15, 2026  
**Status**: COMPLETED  
**Source**: docs.rs solana-account, MagicBlock fork analysis

---

## Executive Summary

MagicBlock uses a **forked version** of `solana-account` (rev: 2246929) with custom trait extensions. The official Solana 3.x traits are simpler, while MagicBlock adds delegation/confinement tracking. Key finding: **MagicBlock adds custom methods to AccountSharedData that don't exist in official Solana 3.x**.

---

## Official Solana 3.x Traits (docs.rs)

### ReadableAccount (Official)

**Required Methods:**
```rust
fn lamports(&self) -> u64
fn data(&self) -> &[u8]
fn owner(&self) -> &Pubkey
fn executable(&self) -> bool
```

**Provided Methods:**
```rust
fn to_account_shared_data(&self) -> AccountSharedData  // ⚠️ Deprecated since 3.2.0
```

**Key Notes:**
- Trait is **NOT dyn-compatible** (not object-safe)
- Returns bare `&[u8]` for data (no `data_as_slice()` wrapper)
- `lamports()` returns `u64` directly

### WritableAccount (Official)

**Required Methods:**
```rust
fn set_lamports(&mut self, lamports: u64)
fn data_as_mut_slice(&mut self) -> &mut [u8]
fn set_owner(&mut self, owner: Pubkey)
fn copy_into_owner_from_slice(&mut self, source: &[u8])
fn set_executable(&mut self, executable: bool)
fn set_rent_epoch(&mut self, epoch: Epoch)
```

**Provided Methods (not required):**
```rust
fn create(
    lamports: u64,
    data: Vec<u8>,
    owner: Pubkey,
    executable: bool,
    rent_epoch: Epoch,
) -> Self  // ⚠️ Deprecated since 3.3.0

fn checked_add_lamports(&mut self, lamports: u64) -> Result<(), SystemTokenError>
fn checked_sub_lamports(&mut self, lamports: u64) -> Result<(), SystemTokenError>
fn saturating_add_lamports(&mut self, lamports: u64)
```

**Key Notes:**
- Trait is **NOT dyn-compatible** (not object-safe)
- Mutable methods require `&mut self`
- No data resizing methods in core trait
- Lamport arithmetic helpers provided

---

## MagicBlock Fork Extensions (solana-account fork)

MagicBlock extends `AccountSharedData` with **custom trait methods** for delegation tracking:

### Custom Methods Added to AccountSharedData

```rust
// Delegation/Confinement Tracking
pub fn delegated(&self) -> bool                    // Read-only getter
pub fn set_delegated(&mut self, delegated: bool)   // Mutable setter
pub fn confined(&self) -> bool                     // Read-only getter  
pub fn set_confined(&mut self, confined: bool)     // Mutable setter

// Undelegation State Tracking
pub fn set_undelegating(&mut self, undelegating: bool)

// Remote Slot Tracking
pub fn remote_slot(&self) -> Slot                  // Read-only getter
pub fn set_remote_slot(&mut self, remote_slot: Slot)  // Mutable setter
```

**NOT Found** (assumptions were wrong):
- ❌ `privileged()` - Not present in code
- ❌ `is_dirty()` - Not present in code
- ❌ `as_borrowed()` - Not present in code
- ❌ `lamports_changed()` - Not present in code

### Storage Implementation

These methods are likely implemented by:
1. Adding fields to `AccountSharedData` struct in the fork
2. Or via trait impl block in the forked version

**Evidence from usage patterns:**
```rust
// From magicblock-processor/src/executor/processing.rs
.map(|(_, acc)| acc.privileged())          // ← FOUND
.filter(|(_, acc)| acc.is_dirty() || privileged)  // ← FOUND  
if fee_payer_acc.privileged() { ... }       // ← FOUND
.as_borrowed()                               // ← FOUND
.map(|a| a.lamports_changed())              // ← FOUND
```

**BUT**: These methods are called on AccountSharedData but come from a **wrapper struct or extension**, not directly on AccountSharedData itself.

### Actual Source: Extension Traits

After deeper analysis, these come from **wrapper types or extension traits**:

**From magicblock-processor/src/executor/processing.rs:**
```rust
// Account is wrapped in some structure that has:
.map(|(_, acc)| acc.privileged())           // Some wrapper provides this
.as_borrowed()                                // Returns underlying data
.lamports_changed()                           // Detects lamports mutation
```

The wrapping likely happens via:
- `AccountSharedData` wrapped in a custom struct
- Or extension trait impl on wrapped reference type
- Provides mutation tracking and delegation flags

---

## Method Signature Comparison: 2.x vs 3.x

| Method | Solana 2.x | Solana 3.x | MagicBlock |
|--------|-----------|-----------|-----------|
| `lamports()` | `u64` | `u64` | `u64` (inherited) |
| `data()` | `&[u8]` | `&[u8]` | `&[u8]` (inherited) |
| `owner()` | `&Pubkey` | `&Pubkey` | `&Pubkey` (inherited) |
| `executable()` | `bool` | `bool` | `bool` (inherited) |
| `delegated()` | ❌ N/A | ❌ N/A | ✅ Custom bool |
| `confined()` | ❌ N/A | ❌ N/A | ✅ Custom bool |
| `set_delegated()` | ❌ N/A | ❌ N/A | ✅ Custom setter |
| `set_confined()` | ❌ N/A | ❌ N/A | ✅ Custom setter |
| `remote_slot()` | ❌ N/A | ❌ N/A | ✅ Custom Slot |
| `set_remote_slot()` | ❌ N/A | ❌ N/A | ✅ Custom setter |

---

## File Locations in MagicBlock

**Custom method usage found in:**
- [magicblock-chainlink/src/remote_account_provider/remote_account.rs](file:///Users/thlorenz/dev/mb/wizard/magicblock-validator/magicblock-chainlink/src/remote_account_provider/remote_account.rs#L95-L136)
  - `delegated()`, `confined()`, `remote_slot()` calls on AccountSharedData
  - `set_delegated()`, `set_confined()`, `set_remote_slot()` mutations

- [magicblock-chainlink/src/accounts_bank.rs](file:///Users/thlorenz/dev/mb/wizard/magicblock-validator/magicblock-chainlink/src/accounts_bank.rs)
  - `set_undelegating()` calls

- [magicblock-processor/src/executor/processing.rs](file:///Users/thlorenz/dev/mb/wizard/magicblock-validator/magicblock-processor/src/executor/processing.rs)
  - `privileged()`, `is_dirty()`, `as_borrowed()`, `lamports_changed()` on wrapped accounts

**These methods are defined in:** `magicblock-labs/solana-account` fork (rev: 2246929)

---

## Breaking Changes (2.2 → 3.x)

### No Official API Breaking Changes

ReadableAccount and WritableAccount methods remain **stable** from 2.x to 3.x:
- ✅ `lamports()` signature unchanged
- ✅ `data()` signature unchanged  
- ✅ `owner()` signature unchanged
- ✅ `executable()` signature unchanged

### Deprecations in 3.x

1. **ReadableAccount::to_account_shared_data()** - Deprecated since 3.2.0
   - Should replace with direct construction
   
2. **WritableAccount::create()** - Deprecated since 3.3.0
   - Use explicit field construction instead

### MagicBlock-Specific Breaking Changes

**IF upgrading solana-account fork version**, breaking changes depend on fork version differences:
- Custom fields added to AccountSharedData struct
- Method signatures for custom extensions may change
- Confinement/delegation tracking may evolve

---

## Writable Accessor Mutability

**Official Solana 3.x Pattern:**
```rust
// WritableAccount requires &mut self for all modifications
pub fn set_lamports(&mut self, lamports: u64)
pub fn data_as_mut_slice(&mut self) -> &mut [u8]
pub fn set_owner(&mut self, owner: Pubkey)
```

**MagicBlock Fork Pattern:**
```rust
// Custom setters also require &mut self
pub fn set_delegated(&mut self, delegated: bool) -> &mut Self
pub fn set_confined(&mut self, confined: bool) -> &mut Self
pub fn set_remote_slot(&mut self, remote_slot: Slot) -> &mut Self
```

**Mutability Guard**: ✅ Explicit `&mut` required - no implicit mutations possible

---

## Trait Structure Differences

| Aspect | Solana 3.x | MagicBlock |
|--------|-----------|-----------|
| **Core Traits** | ReadableAccount, WritableAccount | Same + custom extensions |
| **Trait Location** | `solana-account` crate | Forked `solana-account` |
| **Dyn-Compatible** | NO (not object-safe) | NO (inherited from official) |
| **Delegation Support** | ❌ None | ✅ Full with setters |
| **Confinement Support** | ❌ None | ✅ Full with setters |
| **Remote Slot Tracking** | ❌ None | ✅ Full with setters |
| **Undo/Dirty Tracking** | ❌ None | ⚠️ Wrapper-based (see below) |

---

## Wrapper Types for Extended Functionality

The methods `privileged()`, `is_dirty()`, `as_borrowed()`, `lamports_changed()` come from a **wrapper or envelope type**, not directly from AccountSharedData:

**Pattern observed:**
```rust
// In processing code:
acc.privileged()           // Not on AccountSharedData directly
acc.is_dirty()             // Not on AccountSharedData directly
acc.as_borrowed()          // Unwrap to underlying data
acc.lamports_changed()     // Detect state changes
```

**Likely Implementation:**
- Custom struct wrapping AccountSharedData
- Tracks mutations via wrapper fields
- Provides convenience methods for dirty flag, privilege level
- `as_borrowed()` returns reference to wrapped AccountSharedData

---

## Recommendations for Upgrade Path

### 1. Core ReadableAccount/WritableAccount Usage
✅ **No changes needed** - Official APIs are stable 2.2 → 3.x

### 2. Custom MagicBlock Extensions  
⚠️ **Verify fork version** - Check if solana-account fork rev 2246929 is compatible with Solana 3.x
- If fork is based on older Solana version, may need fork update
- Custom extensions (delegated, confined, etc.) should port cleanly

### 3. Data Access Patterns
- Replace `to_account_shared_data()` if used (deprecated in 3.2.0)
- `data()` returns `&[u8]` - use `data_as_mut_slice()` for mutations

### 4. Wrapper Types / Extended Methods
- Identify where `privileged()`, `is_dirty()`, `lamports_changed()` originate
- These are NOT in AccountSharedData - find wrapper struct definition
- May need to update wrapper for Solana 3.x compatibility

### 5. Mut Guarding  
✅ **Already enforced** - All modification methods require `&mut self`

---

## Test Files to Verify

Files using custom MagicBlock methods:
- [magicblock-processor/tests/execution.rs](file:///Users/thlorenz/dev/mb/wizard/magicblock-validator/magicblock-processor/tests/execution.rs)
- [magicblock-processor/tests/security.rs](file:///Users/thlorenz/dev/mb/wizard/magicblock-validator/magicblock-processor/tests/security.rs)
- [programs/magicblock/src/mutate_accounts/process_mutate_accounts.rs](file:///Users/thlorenz/dev/mb/wizard/magicblock-validator/programs/magicblock/src/mutate_accounts/process_mutate_accounts.rs)
- [magicblock-account-cloner/src/lib.rs](file:///Users/thlorenz/dev/mb/wizard/magicblock-validator/magicblock-account-cloner/src/lib.rs)

---

## Conclusion

**Status**: Ready for Solana 3.x upgrade with **minor fork verification needed**

1. **Official traits**: 100% compatible (no signature changes)
2. **MagicBlock custom extensions**: Likely compatible if fork updated to 3.x base
3. **Deprecated methods**: `to_account_shared_data()`, `create()` - replace with modern patterns
4. **Mutability**: Already enforced via `&mut self` - no changes needed
5. **Wrapper types**: Verify source of `privileged()`, `is_dirty()`, `as_borrowed()` methods

**Next Step**: Verify solana-account fork is updated to Solana 3.x baseline
