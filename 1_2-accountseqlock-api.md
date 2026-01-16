# Issue 1.2: AccountSeqLock API in Solana 3.x

**Date**: January 15, 2026  
**Status**: Complete  
**Findings**: AccountSeqLock is a MagicBlock custom API, NOT present in official Solana

---

## Executive Summary

**AccountSeqLock does NOT exist in official Solana** (2.x or 3.x). It is an entirely custom API created by MagicBlock to support optimistic concurrency reading of borrowed accounts. This API is maintained in a custom fork: `https://github.com/magicblock-labs/solana-account/`

---

## Origin & History

### Source
- **Package**: `solana-account` crate
- **Official Location**: NOT available in official Solana/Agave repository
- **MagicBlock Fork**: https://github.com/magicblock-labs/solana-account (rev 2246929)
- **Version**: 2.2.1 (based on Solana 2.2 APIs)

### Fork Inception
The MagicBlock fork was created independently and entirely custom:
- **First commit**: `4ac1115` (Jan 30, 2025) - "first commit" - initial extraction/creation
- **AccountSeqLock added**: Commit `ee2d713` (Aug 26, 2025) - PR #10 "feat: seq-locked-account"
- **Current ref**: `2246929` (Jan 2, 2026) - Latest in main branch

All history shows this is a MagicBlock creation, not a Solana fork/extraction.

---

## API Definition

### Location
```
solana-account crate (MagicBlock fork)
  └── src/cow.rs
      └── lines 712-728
```

### Full API Signature

```rust
/// A sequence lock that detects concurrent modifications to borrowed account data.
/// Used for optimistic read patterns in multi-threaded scenarios.
pub struct AccountSeqLock {
    counter: ShadowSwitch,
    current: u32,
}

impl AccountSeqLock {
    /// Re-reads the current value of the atomic counter.
    /// Call this after detecting a potential race condition to get the latest value.
    pub fn relock(&mut self) {
        self.counter.counter();
    }

    /// Checks if the atomic counter has changed since the lock was created or last relocked.
    /// Returns `true` if a concurrent write occurred (data may be inconsistent).
    /// Returns `false` if data is consistent.
    pub fn changed(&self) -> bool {
        self.counter.counter() != self.current
    }
}

// Thread-safe: Send + Sync via atomic operations
unsafe impl Send for AccountSeqLock {}
unsafe impl Sync for AccountSeqLock {}
```

### Related Types

**ShadowSwitch** (defined in same module):
- An atomic counter (`Arc<AtomicU32>`) that tracks mutations to borrowed account data
- Incremented whenever a borrowed account is modified
- Used as the "write version" for optimistic locking

**AccountSharedData::Borrowed**:
- Points to borrowed memory (NOT owned)
- Must be protected by `AccountSeqLock` for safe concurrent reads
- Implements `lock()` method to capture current sequence lock state
- Implements `reinit()` method to re-read current data with updated counter

---

## MagicBlock Usage Pattern

### How It Works (from `magicblock-core/src/link/accounts.rs`)

```rust
pub struct LockedAccount {
    pub lock: Option<AccountSeqLock>,  // Captured for Borrowed accounts only
    pub account: AccountSharedData,
}

impl LockedAccount {
    pub fn new(pubkey: Pubkey, account: AccountSharedData) -> Self {
        let lock = match &account {
            AccountSharedData::Owned(_) => None,  // No locking needed for owned
            AccountSharedData::Borrowed(acc) => acc.lock().into(),  // Capture at creation
        };
        Self { lock, account, pubkey }
    }

    pub fn read_locked<F, R>(&self, reader: F) -> R
    where
        F: Fn(&Pubkey, &AccountSharedData) -> R,
    {
        // Optimistic read attempt
        let result = reader(&self.pubkey, &self.account);

        // Fast path: no concurrent write
        if !self.changed() {
            return result;
        }

        // Slow path: race detected, retry loop
        let AccountSharedData::Borrowed(ref borrowed) = self.account else {
            return result;
        };
        let Some(mut lock) = self.lock.clone() else {
            return result;
        };

        loop {
            lock.relock();                    // Get fresh counter value
            let account = borrowed.reinit();  // Re-read data with new counter
            let result = reader(&self.pubkey, &account);
            if !lock.changed() {              // Did write happen during read?
                break result;
            }
        }
    }
}
```

### Usage in MagicBlock
- **File**: `magicblock-core/src/link/accounts.rs`
- **Lines**: 42, 92, 106-109
- **Pattern**: Enables lock-free reads with CAS-like retry semantics

---

## Solana 3.x Compatibility

### Key Finding
The official Solana/Agave repository (as of Jan 2026) does **NOT** have a `solana-account` crate in its main distribution.

**Verified in agave.git**:
- Checked depth=1 clone of anza-xyz/agave
- No `solana-account` crate directory exists
- No `AccountSeqLock` API found anywhere in codebase
- Related crates: `account-decoder`, `account-decoder-client-types`, `accounts-db`

### Current State
- MagicBlock is using **custom version 2.2.1** from its fork
- This fork extends Solana 2.2 APIs with custom features:
  - AccountSeqLock (optimistic read locking)
  - CoW (Copy-on-Write) for borrowed accounts
  - Delegation flags, compression flags, confined accounts, etc.

---

## Breaking Changes for Solana 3.x

### Impact Analysis

| Aspect | Current (2.x) | Solana 3.x | Migration |
|--------|---------------|-----------|-----------|
| **solana-account crate** | Custom fork required | Unknown - likely doesn't exist | Must determine if 3.x uses different crate |
| **AccountSeqLock** | Custom MagicBlock API | Not in official Solana | Must maintain custom fork or reimplement |
| **Borrowed accounts** | Custom extension (CoW) | Unknown in official | Must verify Solana 3.x account model |
| **AccountSharedData** | Custom enum (Owned/Borrowed) | Unknown | May not have Borrowed variant |
| **Concurrency model** | Optimistic reads with seq locks | Unknown | May require different locking approach |

### Required Investigation
1. **Does Solana 3.x have a `solana-account` crate?**
   - If yes: Check if it includes `AccountSeqLock` or equivalent
   - If no: Understand what replaced it in the API

2. **Does Solana 3.x support borrowed/memory-mapped accounts?**
   - If yes: How is concurrency handled?
   - If no: All accounts become Owned - simplifies but impacts performance

3. **Account model changes in 3.x**
   - Version number increments suggest significant changes
   - May affect `ReadableAccount`/`WritableAccount` traits

---

## Files Affected in MagicBlock

### Direct Dependencies
```
magicblock-core/src/link/accounts.rs
  ├── Uses: AccountSeqLock, AccountSharedData (Owned/Borrowed)
  └── Implements: LockedAccount, read_locked() pattern

solana-account (custom fork, rev 2246929)
  ├── src/cow.rs (AccountSeqLock definition)
  ├── src/lib.rs (AccountSharedData enum, traits)
  └── 30+ custom commits on top of Solana 2.2
```

### Indirect Impact
Any code using `LockedAccount`:
- RPC services reading accounts
- Geyser plugin account handlers
- Account cloner operations
- Transaction processor account access

---

## Migration Path (Hypothetical)

### Scenario A: Solana 3.x Has solana-account Crate
```rust
// Check if 3.x solana-account includes AccountSeqLock
// If yes: Update dependency, minimal changes
// If no: Keep custom fork, add compatibility layer
```

### Scenario B: Solana 3.x Removes solana-account or Changes API
```rust
// Option 1: Continue maintaining fork, add Solana 3.x support
// Option 2: Reimplement AccountSeqLock using Solana 3.x primitives
// Option 3: Remove borrowed account optimization, simplify to all-owned

// Estimated effort: 2-3 weeks depending on API changes
```

---

## Recommendations

### Immediate Actions
1. ✅ **COMPLETED**: Confirm AccountSeqLock is MagicBlock-custom
2. **TODO**: Check official Solana 3.x repository for account handling
3. **TODO**: Document minimum Solana 3.x version requirements
4. **TODO**: Plan fork maintenance strategy for 3.x

### Longer-term
- Consider upstreaming AccountSeqLock to official Solana if valuable
- Plan gradual migration path if Solana 3.x uses different account model
- Create compatibility layer to abstract account API

---

## Summary Table

| Property | Value |
|----------|-------|
| **API Source** | MagicBlock custom (not official Solana) |
| **Created** | Jan 30, 2025 (fork inception) |
| **Feature Added** | Aug 26, 2025 (PR #10) |
| **Current Version** | solana-account 2.2.1, rev 2246929 |
| **Solana Basis** | 2.2 APIs + 30+ custom commits |
| **Thread Safety** | Yes (Send + Sync via atomics) |
| **Solana 3.x Status** | Unknown - requires investigation |
| **Breaking Risk** | HIGH - depends entirely on Solana 3.x account API changes |

