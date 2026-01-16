# Research Issue 4.2: Instruction and CompiledInstruction Changes in Solana 3.x

**Date**: January 15, 2026  
**Status**: Complete  
**Target**: Solana SDK 2.x → 3.x Migration  

---

## Executive Summary

Solana 3.x introduces breaking changes to the `Instruction` and `CompiledInstruction` types that impact message parsing and instruction manipulation code. The most significant changes are:

1. **Type System**: `Pubkey` is now a type alias for `Address` (renamed, more semantically correct)
2. **Serialization**: No breaking changes to serialization format, but API methods may differ
3. **Module Organization**: Many instruction-related types moved to separate crates
4. **Method Removals**: Some helper methods removed as part of SDK reorganization

---

## Part 1: Instruction Struct Definition

### Solana 2.x (solana-instruction 2.2)

```rust
pub struct Instruction {
    /// Pubkey of the program that executes this instruction.
    pub program_id: Pubkey,
    /// Metadata describing accounts that should be passed to the program.
    pub accounts: Vec<AccountMeta>,
    /// Opaque data passed to the program for its own interpretation.
    pub data: Vec<u8>,
}
```

**Type**: Located in `solana_program::instruction` or `solana_instruction` crate  
**Field Types**:
- `program_id`: `Pubkey` (32 bytes, Ed25519 public key)
- `accounts`: `Vec<AccountMeta>` (variable-length array of account metadata)
- `data`: `Vec<u8>` (arbitrary instruction data bytes)

### Solana 3.x (solana-instruction 3.0+)

```rust
pub struct Instruction {
    /// Pubkey of the program that executes this instruction.
    pub program_id: Address,  // Type alias for the new Address type
    /// Metadata describing accounts that should be passed to the program.
    pub accounts: Vec<AccountMeta>,
    /// Opaque data passed to the program for its own interpretation.
    pub data: Vec<u8>,
}
```

**Breaking Change**: The field type changed from `Pubkey` to `Address`

### Key Difference: Pubkey ↔ Address

| Aspect | Solana 2.x | Solana 3.x |
|--------|-----------|-----------|
| **Type Name** | `Pubkey` | `Address` (primary), `Pubkey` (alias) |
| **Semantic Clarity** | Generic "public key" | Explicitly named "address" |
| **Underlying Structure** | Same (32 bytes) | Same (32 bytes) |
| **Compatibility** | Uses `Pubkey` throughout | Uses `Address` throughout, `Pubkey = Address` |
| **Byte Representation** | Private fields | Private fields |
| **Access Method** | Direct field access | Uses `Address::as_bytes()` |

---

## Part 2: CompiledInstruction Structure

### Solana 2.x (in solana_message)

```rust
#[derive(Serialize, Deserialize, Debug, Clone, Eq, PartialEq)]
pub struct CompiledInstruction {
    pub program_id_index: u8,          // Index into message's account_keys
    pub accounts: Vec<u8>,              // Indices into message's account_keys
    pub data: Vec<u8>,                  // Instruction data bytes
}
```

**Purpose**: Compiled form of `Instruction` used in transaction messages
- Stores **indices** rather than public keys (saves space)
- `program_id_index`: Single u8 pointing to program account in message
- `accounts`: Vec of u8 indices pointing to accounts in message account_keys
- `data`: Same as `Instruction.data`

### Solana 3.x (in solana_message)

```rust
#[derive(Serialize, Deserialize, Debug, Clone, Eq, PartialEq)]
pub struct CompiledInstruction {
    pub program_id_index: u8,          // No change
    pub accounts: Vec<u8>,              // No change
    pub data: Vec<u8>,                  // No change
}
```

**No Breaking Changes**: The structure itself is unchanged  
**BUT**: Methods and associated functions may have changed

---

## Part 3: Method Signature Changes

### Instruction Methods

#### Solana 2.x Key Methods
```rust
impl Instruction {
    // Creator functions
    pub fn new_with_borsh_data(
        program_id: Pubkey,
        data: &impl BorshSerialize,
        accounts: Vec<AccountMeta>,
    ) -> Self { ... }
    
    pub fn new_with_bincode_data(
        program_id: Pubkey,
        data: &impl Serialize,
        accounts: Vec<AccountMeta>,
    ) -> Self { ... }
    
    // Accessors
    pub fn program_id(&self) -> &Pubkey { ... }
    pub fn accounts(&self) -> &[AccountMeta] { ... }
    pub fn data(&self) -> &[u8] { ... }
}
```

#### Solana 3.x Key Methods
```rust
impl Instruction {
    // Creator functions - SAME SIGNATURE (if Address ≈ Pubkey)
    pub fn new_with_borsh_data(
        program_id: Address,  // Changed type parameter
        data: &impl BorshSerialize,
        accounts: Vec<AccountMeta>,
    ) -> Self { ... }
    
    pub fn new_with_bincode_data(
        program_id: Address,  // Changed type parameter
        data: &impl Serialize,
        accounts: Vec<AccountMeta>,
    ) -> Self { ... }
    
    // Accessors - SAME (return type changed to Address)
    pub fn program_id(&self) -> &Address { ... }  // Return type
    pub fn accounts(&self) -> &[AccountMeta] { ... }
    pub fn data(&self) -> &[u8] { ... }
}
```

**Breaking**: Any code expecting `Pubkey` will get type mismatch errors with `Address`

### CompiledInstruction Methods

No significant method changes documented. The type is primarily used for serialization/deserialization.

---

## Part 4: Serialization/Deserialization APIs

### 2.x Serialization
```rust
// Bincode serialization (used for transaction encoding)
let bytes = bincode::serialize(&instruction)?;
let instruction: Instruction = bincode::deserialize(&bytes)?;

// Borsh support (alternative)
let bytes = instruction.try_to_vec()?;
let instruction = Instruction::try_from_slice(&bytes)?;
```

### 3.x Serialization
```rust
// Same API, but type parameters changed to Address
// No changes to serialization format
let bytes = bincode::serialize(&instruction)?;
let instruction: Instruction = bincode::deserialize(&bytes)?;

// Borsh support unchanged
let bytes = instruction.try_to_vec()?;
let instruction = Instruction::try_from_slice(&bytes)?;
```

**Key Point**: **Binary serialization format is unchanged** - only the Rust types differ

---

## Part 5: Module Organization Changes

### Removed from solana-sdk/solana-program 3.x

The following instruction-related modules were extracted to separate crates:

| Module (2.x) | New Location (3.x) | New Crate |
|--------------|-------------------|-----------|
| `system_instruction` | `solana_system_interface::instruction` | `solana-system-interface` v2+ |
| `system_program` | `solana_system_interface::program` | `solana-system-interface` v2+ |
| `loader_upgradeable_instruction` | `solana_loader_v3_interface::instruction` | `solana-loader-v3-interface` |
| `loader_v4_instruction` | `solana_loader_v4_interface::instruction` | `solana-loader-v4-interface` |
| `vote` | `solana_vote_interface` | `solana-vote-interface` |
| `stake` / `stake_history` | `solana_stake_interface` | `solana-stake-interface` |

### New Crate Imports Required

```toml
# Cargo.toml - Solana 3.x
[dependencies]
solana-instruction = "3.0"
solana-system-interface = { version = "2", features = ["bincode"] }
solana-loader-v3-interface = "3.0"
```

---

## Part 6: Address Type Introduction

### The Address Type (NEW in Solana 3.x)

The new `Address` type is a **semantic wrapper** around the 32-byte Solana address, providing:

1. **Better naming**: `Address` is clearer than `Pubkey`
2. **Type alias compatibility**: `Pubkey = Address` for backward compatibility
3. **Same underlying representation**: Still 32 bytes of Ed25519 public key data

### Usage Pattern

```rust
// Solana 2.x
use solana_pubkey::Pubkey;
let program_id: Pubkey = ...;

// Solana 3.x (compatible approach)
use solana_pubkey::Address;
let program_id: Address = ...;

// Or using the alias
use solana_pubkey::Pubkey;  // Still works, just an alias now
let program_id: Pubkey = ...; // Actually Address under the hood
```

### Type Mismatch Resolution

When dependency versions don't match (one uses Address, one uses Pubkey):

```
error[E0308]: mismatched types
  expected: `solana_pubkey::Address`
     found: `solana_pubkey::Pubkey`

note: these types are equivalent, but they are not equal
types differ in mutability modifiers
```

**Solution**: Ensure all dependencies upgraded to solana-sdk >= 3.0

---

## Part 7: Breaking Changes Identified

### 1. **Type Parameter Changes** (HIGH IMPACT)
- **Issue**: Functions expecting `Pubkey` now receive `Address`
- **Affected Code**: Any function creating/manipulating `Instruction` structs
- **Impact**: Type errors at compile time
- **Example**:
  ```rust
  // 2.x code
  let ix = Instruction {
      program_id: some_pubkey,  // Type: Pubkey
      accounts: vec![],
      data: vec![],
  };
  
  // 3.x requires
  let ix = Instruction {
      program_id: some_address,  // Type: Address (incompatible without conversion)
      accounts: vec![],
      data: vec![],
  };
  ```

### 2. **Module Reorganization** (HIGH IMPACT)
- **Issue**: `system_instruction`, `system_program`, etc. removed from main SDK
- **Affected Code**: Any direct imports like `use solana_sdk::system_instruction`
- **Impact**: Compilation errors (module not found)
- **Example**:
  ```rust
  // 2.x - Works
  use solana_sdk::system_instruction;
  
  // 3.x - Fails
  // error: failed to resolve: could not find `system_instruction` in `solana_sdk`
  
  // 3.x - Correct
  use solana_system_interface::instruction as system_instruction;
  ```

### 3. **InstructionError Changes** (MEDIUM IMPACT)
- **Issue**: `BorshIoError(String)` parameter removed
- **Affected Code**: Error handling for instruction errors
- **Change**: `InstructionError::BorshIoError` no longer takes a String parameter
- **Example**:
  ```rust
  // 2.x
  InstructionError::BorshIoError("error message".to_string())
  
  // 3.x - Signature changed, no string parameter
  InstructionError::BorshIoError  // Unit variant
  ```

### 4. **Message Parsing** (MEDIUM IMPACT)
- **Issue**: Methods for message inspection may have changed or been removed
- **Affected Code**: Code that inspects/manipulates transaction messages
- **Impact**: Some convenience methods from 2.x may not exist in 3.x
- **Mitigation**: May need to manually parse account_keys indices

### 5. **Hash Inner Bytes** (LOW IMPACT)
- **Issue**: Hash inner bytes made private
- **Old Code**: `hash.0` (direct field access)
- **New Code**: `hash.as_bytes()` method
- **Impact**: Compilation errors when accessing hash internals

---

## Part 8: MagicBlock Codebase Impact Analysis

### Current Solana Version
MagicBlock currently uses **Solana 2.2**:
```toml
solana-instruction = { version = "2.2" }
solana-message = { version = "2.2" }
solana-program = "2.2"
```

### High-Impact Files in MagicBlock

#### 1. **magicblock-processor** (CRITICAL)
Located: `/magicblock-processor/src/`

**Files Using Instruction Types**:
- `scheduler/tests.rs` - Creates `Vec<Instruction>` structures
- `scheduler/scheduler.rs` - Instruction manipulation
- `processor.rs` - Core instruction processing

**Breaking Changes**:
```rust
// Current (2.x)
let instructions: Vec<Instruction> = ...;
let ix = Instruction { program_id: pubkey, ... };

// Needs update (3.x)
let instructions: Vec<Instruction> = ...;
let ix = Instruction { program_id: address, ... };  // Type mismatch
```

#### 2. **magicblock-committor-program**
Located: `/magicblock-committor-program/src/processor.rs`

**Impact**: Program-specific instruction creation
**Issue**: If using `system_instruction` or other moved modules

#### 3. **magicblock-magic-program-api**
Located: `/magicblock-magic-program-api/src/instruction.rs`

**Impact**: Instruction serialization/deserialization
**Issue**: If doing custom instruction creation or parsing

#### 4. **test-kit and test-integration**
Located: `/test-kit/src/lib.rs`, `/test-integration/`

**Impact**: Test infrastructure creating transactions
**Issue**: Mock transaction builders will fail type checks

### Dependency Chain Risk

**Chain**: magicblock-validator → magicblock-processor → solana-* crates

If upgrading:
1. solana-instruction 2.2 → 3.0
2. Must also upgrade solana-pubkey, solana-message
3. All transitive dependencies must be 3.x compatible

---

## Part 9: Migration Steps

### Step 1: Update Dependencies

```toml
# Before
solana-instruction = { version = "2.2" }
solana-message = { version = "2.2" }
solana-program = "2.2"

# After
solana-instruction = { version = "3.0" }
solana-message = { version = "3.0" }
solana-program = "3.0"
```

### Step 2: Add New Interface Crates

```toml
[dependencies]
# If using system programs
solana-system-interface = { version = "2", features = ["bincode"] }

# If using vote
solana-vote-interface = { version = "2" }

# If using stake
solana-stake-interface = { version = "2" }
```

### Step 3: Update Imports

**OLD (2.x)**:
```rust
use solana_sdk::system_instruction;
use solana_program::system_instruction;
use solana_program::vote;
```

**NEW (3.x)**:
```rust
use solana_system_interface::instruction as system_instruction;
use solana_system_interface::instruction;
use solana_vote_interface;
```

### Step 4: Fix Type Mismatches

**Pattern**: Replace `Pubkey` with `Address` in function signatures

```rust
// BEFORE
pub fn create_instruction(program_id: Pubkey, ...) -> Instruction {
    Instruction {
        program_id,
        accounts: vec![],
        data: vec![],
    }
}

// AFTER
use solana_pubkey::Address;

pub fn create_instruction(program_id: Address, ...) -> Instruction {
    Instruction {
        program_id,
        accounts: vec![],
        data: vec![],
    }
}
```

### Step 5: Update Error Handling

**Before**:
```rust
match ix_error {
    InstructionError::BorshIoError(msg) => println!("Error: {}", msg),
    _ => {}
}
```

**After**:
```rust
match ix_error {
    InstructionError::BorshIoError => println!("Borsh IO error"),
    _ => {}
}
```

### Step 6: Fix Hash Access

**Before**:
```rust
let bytes = hash.0;  // Direct field access
```

**After**:
```rust
let bytes = hash.as_bytes();  // Method access
```

### Step 7: Compatibility Layer (Optional)

If you need gradual migration, create type aliases:

```rust
// In your crate's lib.rs or prelude
#[cfg(solana_sdk_version = "3")]
use solana_pubkey::Address as Pubkey;

// Now you can use Pubkey in your code and it works for both versions
pub fn your_function(program_id: Pubkey) { ... }
```

### Step 8: Run Tests and Build

```bash
cargo build --all
cargo test --all
make lint
make fmt
```

---

## Part 10: Testing Strategy

### Unit Test Updates

**Before (2.x)**:
```rust
#[test]
fn test_instruction_creation() {
    let pubkey = Pubkey::new_unique();
    let ix = Instruction {
        program_id: pubkey,
        accounts: vec![],
        data: vec![],
    };
    assert_eq!(ix.program_id, pubkey);
}
```

**After (3.x)**:
```rust
#[test]
fn test_instruction_creation() {
    use solana_pubkey::Address;
    let address = Address::new_unique();
    let ix = Instruction {
        program_id: address,
        accounts: vec![],
        data: vec![],
    };
    assert_eq!(ix.program_id, address);
}
```

### Integration Test Coverage

1. **Message Compilation**: Verify `Instruction` → `CompiledInstruction` conversion works
2. **Serialization**: Test instruction serialization roundtrip
3. **Type System**: Ensure no type mismatches in module boundaries
4. **System Program Calls**: If using system_instruction, test with new imports

---

## Part 11: Backward Compatibility Considerations

### Regarding Binary Compatibility

✅ **GOOD NEWS**: Transaction binary format is **unchanged**
- Serialized instructions are identical between 2.x and 3.x
- `CompiledInstruction` binary layout unchanged
- Old transactions can be deserialized with 3.x code

### Regarding Source Compatibility

❌ **SOURCE CODE BREAKS**: 
- Type mismatches due to Address vs Pubkey
- Module reorganization requires import changes
- Some method signatures changed

### Cross-Version Scenarios

If MagicBlock needs to interoperate with both 2.x and 3.x systems:

```rust
// Use feature flags for conditional compilation
#[cfg(feature = "solana-3x")]
use solana_pubkey::Address;

#[cfg(not(feature = "solana-3x"))]
use solana_pubkey::Pubkey as Address;

// Now your code uses Address universally
pub fn compatible_fn(program_id: Address) { ... }
```

---

## Summary Table

| Aspect | Solana 2.x | Solana 3.x | Migration Effort |
|--------|-----------|-----------|-----------------|
| **Instruction struct** | Field: `Pubkey` | Field: `Address` | Medium |
| **CompiledInstruction** | Unchanged structure | Unchanged structure | Low |
| **Serialization** | Bincode/Borsh | Same format | None |
| **system_instruction** | `solana_sdk` | `solana_system_interface` | High |
| **Method signatures** | Use `Pubkey` | Use `Address` | Medium |
| **Error types** | `BorshIoError(String)` | `BorshIoError` | Low |
| **Hash access** | `.0` field | `.as_bytes()` method | Low |
| **Total Migration Complexity** | - | - | **MEDIUM-HIGH** |

---

## References

- **Solana SDK v3 Repository**: https://github.com/anza-xyz/solana-sdk
- **Solana v3 Upgrade Guide**: https://github.com/anza-xyz/solana-sdk#upgrading-from-v2-to-v3
- **Agave v3.0 Release Schedule**: https://github.com/anza-xyz/agave/wiki/v3.0-Release-Schedule
- **Solana Instruction Docs**: https://solana.com/docs/core/instructions
- **Solana Transaction Docs**: https://solana.com/docs/core/transactions

---

## Recommendations for MagicBlock

1. **Staged Migration**: Upgrade one module at a time (processor → committor → API)
2. **Feature Flags**: Use `cfg` attributes for gradual adoption
3. **Test Coverage**: Expand tests for type conversions and module boundaries
4. **Documentation**: Update internal docs regarding `Address` vs `Pubkey` usage
5. **CI/CD**: Add nightly builds against latest solana-sdk
6. **Timeline**: Allocate 1-2 weeks for full migration + testing

---

**Document Generated**: 2026-01-15  
**Status**: Research Complete - Ready for Implementation Planning
