# Issue 3.3: LoaderV4 and BPF Loader Interface Changes in Solana 3.x

**Investigation Date**: January 15, 2026  
**Status**: Verified & Complete

---

## Executive Summary

Solana 3.x maintains **backward compatibility** with LoaderV3 (UpgradeableLoaderState), but **LoaderV4 is now available** as a new alternative loader. Key findings:

- ✅ `UpgradeableLoaderState` enum variants (`Program`, `ProgramData`, `Buffer`) still exist in 3.x
- ✅ `Pubkey::find_program_address()` still works unchanged
- ✅ `bpf_loader_upgradeable` module is stable
- ⚠️ **LoaderV4 introduces a completely different account structure** (not an enum, but a struct)
- ✅ ELF verification and rent calculations remain compatible
- ⚠️ Migration path exists but is **optional** (no forced upgrades in 3.x)

---

## 1. UpgradeableLoaderState Enum (LoaderV3)

### Structure (3.x - Unchanged from 2.x)

```rust
#[derive(Clone, Copy, Debug, Default, Deserialize, Eq, PartialEq, Serialize)]
pub enum UpgradeableLoaderState {
    #[default]
    Uninitialized,
    
    Buffer {
        authority_address: Option<Pubkey>,
    },
    
    Program {
        programdata_address: Pubkey,
    },
    
    ProgramData {
        slot: u64,
        upgrade_authority_address: Option<Pubkey>,
    },
}
```

### Key Properties

| Property | Details |
|----------|---------|
| **Module** | `solana_loader_v3_interface::state::UpgradeableLoaderState` |
| **Serialization** | Binary via `bincode` |
| **Discriminator** | Enum variant discriminator is part of serialized data |
| **Account Owner** | `BPFLoaderUpgradeab1e11111111111111111111111` (unchanged) |
| **Variants** | 4 variants: Uninitialized, Buffer, Program, ProgramData |

### Backward Compatibility Status

✅ **FULLY COMPATIBLE** with Solana 3.x - no changes to API or structure.

MagicBlock's current usage:
```rust
// Still valid in 3.x
let data = bincode::serialize(&UpgradeableLoaderState::ProgramData {
    slot: 0,
    upgrade_authority_address: Some(Pubkey::default()),
})?;
```

---

## 2. BPF Loader Module API

### Core Functions (3.x - Stable)

| Function | Status | Notes |
|----------|--------|-------|
| `bpf_loader_upgradeable::id()` | ✅ Stable | Returns loader program ID |
| `Pubkey::find_program_address()` | ✅ Stable | ProgramData derivation unchanged |
| `create_buffer()` | ✅ Stable | Initialize buffer accounts |
| `deploy_with_max_program_len()` | ✅ Stable | Deploy programs |
| `write()` | ✅ Stable | Write ELF data to buffer |
| `upgrade()` | ✅ Stable | Upgrade programs |
| `set_upgrade_authority()` | ✅ Stable | Transfer authority |
| `extend_program()` | ✅ Stable | Extend program data |
| `close()` / `close_any()` | ✅ Stable | Close accounts |

### Instruction Types (LoaderV3)

```rust
pub enum UpgradeableLoaderInstruction {
    InitializeBuffer,
    Write { offset: u32, bytes: Vec<u8> },
    DeployWithMaxDataLen { max_data_len: usize },
    Upgrade,
    SetAuthority,
    Close,
    ExtendProgram { additional_bytes: u32 },
    SetAuthorityChecked,
    Migrate,  // ← NEW: Migrate to LoaderV4
    ExtendProgramChecked { additional_bytes: u32 },
}
```

**⚠️ NEW in 3.x**: `Migrate` instruction - optional migration path to LoaderV4.

---

## 3. LoaderV4 API Changes

### Account Structure (NEW)

**LoaderV4 is fundamentally different** - it's a struct, not an enum:

```rust
#[derive(Clone, Copy, Debug, Serialize, Deserialize, Eq, PartialEq)]
pub struct LoaderV4State {
    /// Slot in which the program was last deployed, retracted or initialized.
    pub slot: u64,
    
    /// Address of signer which can send program management instructions 
    /// when the status is not finalized.
    /// Otherwise a forwarding to the next version of the finalized program.
    pub authority_address_or_next_version: Pubkey,
    
    /// Deployment status.
    pub status: LoaderV4Status,
}

#[derive(Clone, Copy, Debug, Serialize, Deserialize, Eq, PartialEq, PartialOrd, Ord)]
#[repr(u8)]
pub enum LoaderV4Status {
    Retracted = 0,   // Program in maintenance mode
    Deployed = 1,    // Program ready to execute
    Finalized = 2,   // Program ready & immutable
}
```

### Key Differences: LoaderV3 vs LoaderV4

| Aspect | LoaderV3 | LoaderV4 |
|--------|----------|---------|
| **Account Owner** | `BPFLoaderUpgradeab1...` | `LoaderV411111111111...` |
| **State Type** | Enum (4 variants) | Struct (always single state) |
| **Account Layout** | Header + ELF in same account | Separate state structure |
| **Program Lifecycle** | Deploy → Upgrade → Close | Retracted → Deployed → Finalized |
| **Authority Handling** | Separate authority field | Combined authority_or_next_version |
| **ProgramData Indirection** | 2-account pair (Program + ProgramData) | Single account per program |

### LoaderV4 Instruction Set

```rust
pub enum LoaderV4Instruction {
    Write { offset: u32, bytes: Vec<u8> },
    Copy { destination_offset: u32, source_offset: u32, length: u32 },
    SetProgramLength { new_size: u32 },
    Deploy,
    Retract,
    TransferAuthority,
    Finalize,
}
```

### LoaderV4 Program ID

```rust
pub const ID: Pubkey = pubkey!("LoaderV411111111111111111111111111111111111");
```

---

## 4. Breaking Changes Analysis

### For LoaderV3 Users (Current MagicBlock Usage)

✅ **NO BREAKING CHANGES** - Solana 3.x fully supports LoaderV3.

Your current code will work:
```rust
use solana_loader_v3_interface::state::UpgradeableLoaderState;

let data = bincode::serialize(&UpgradeableLoaderState::ProgramData {
    slot: deploy_slot,
    upgrade_authority_address: Some(authority),
})?;

let (program_data_address, _) = Pubkey::find_program_address(
    &[program_id.as_ref()],
    &bpf_loader_upgradeable::id(),
);
```

### For LoaderV4 Adoption

⚠️ **OPTIONAL MIGRATION** - LoaderV4 is available but not required in Solana 3.x.

Key differences if migrating:

1. **State Structure Change**: From enum to struct
2. **Account Layout**: Single account vs. Program+ProgramData pair
3. **Instruction Set**: New instruction types, different workflow
4. **Authority Field**: Combined authority_or_next_version
5. **Status States**: Different state machine (Retracted/Deployed/Finalized)

---

## 5. ELF Loading and Verification

### ELF Verification API (3.x - Unchanged)

✅ **STABLE** - No breaking changes to ELF loading/verification:

- ELF format validation occurs at deployment
- Verification hooks unchanged
- Program cache invalidation still works
- SBPFv1 support maintained (programs must use compatible bytecode)

### Program Cache Interaction

✅ **STABLE** - Cache update mechanism unchanged:

- After deployment/upgrade, loader notifies program cache
- Cache invalidation still uses account state changes
- Slot tracking unchanged
- Executable flag handling unchanged

### Rent Calculations

✅ **STABLE** - Rent exemption calculations unchanged:

```rust
let min_balance = rent.minimum_balance(data.len());
```

Works identically for both LoaderV3 and LoaderV4 accounts.

---

## 6. Migration Considerations

### LoaderV3 → LoaderV4 Path

**Status in Solana 3.x**: Available but **NOT mandatory**

Available instruction:
- `UpgradeableLoaderInstruction::Migrate` - Migrate a LoaderV3 program to LoaderV4

**Timeline**:
1. **Phase 1** (Current): Solana 3.x - Both loaders coexist
2. **Phase 2**: LoaderV4 as on-chain program (not builtin)
3. **Phase 3**: Optional migration research (may force migration later)

**Current Status**: Programs can opt-in to migrate but **must be prepared** to support both LoaderV3 and LoaderV4 formats.

### Recommended Approach for MagicBlock

1. **Continue using LoaderV3** (safer, proven, no migration pressure in 3.x)
2. **Add LoaderV4 support** when:
   - SIMDs establish clear migration timeline
   - Your deployment tooling is LoaderV4-ready
   - You need LoaderV4-specific features
3. **Monitor Solana roadmap** for migration announcements

---

## 7. MagicBlock Code Impact Assessment

### Files Using Loader APIs

| File | Usage | Impact |
|------|-------|--------|
| `magicblock-processor/src/loader.rs` | LoaderV3 ProgramData/Program state creation | ✅ No changes needed |
| `magicblock-account-cloner/src/bpf_loader_v1.rs` | LoaderV3 state serialization | ✅ No changes needed |
| `magicblock-chainlink/src/remote_account_provider/program_account.rs` | LoaderV3 state parsing | ✅ No changes needed |
| `programs/magicblock/src/schedule_transactions/` | bpf_loader_upgradeable references | ✅ No changes needed |

### Required Changes

✅ **NONE for Solana 3.x compatibility**

All current code remains valid and functional.

---

## 8. Upgrade Recommendations

### Immediate Actions

1. ✅ **No code changes required** - Your LoaderV3 code is fully compatible
2. ✅ **Update dependencies** to Solana 3.x stable versions:
   - `solana-program` 3.x
   - `solana-loader-v3-interface` 6.x+
   - `solana-sdk-ids` 3.x

### Future Preparation (When LoaderV4 Becomes Mandatory)

1. **Add LoaderV4 support** when migration timeline is announced
2. **Dual-support pattern**:
   ```rust
   // Pseudo-code
   match program_owner {
       bpf_loader_upgradeable::ID => {
           // Handle LoaderV3
           let state = UpgradeableLoaderState::try_from(&account_data)?;
       },
       loader_v4::ID => {
           // Handle LoaderV4
           let state = LoaderV4State::try_from(&account_data)?;
       },
       _ => Err("Unknown loader"),
   }
   ```
3. **Test both loader formats** in your integration tests

### Dependencies to Monitor

```toml
[dependencies]
solana-program = "3.x"  # Includes both V3 and V4 interfaces
solana-loader-v3-interface = "6.x"
solana-loader-v4-interface = "3.x"  # New, optional for now
solana-sdk-ids = "3.x"
```

---

## 9. Known Issues & Caveats

### LoaderV3 Limitations (Not Fixed in 3.x)

From RFC-0423, these issues remain in LoaderV3:

1. ⚠️ **Account Indirection**: Program + ProgramData pair creates indirection
2. ⚠️ **Metadata/ELF Coupling**: Slot and authority stored with ELF data
3. ⚠️ **ProgramData Greedy Lamport Locking**: Zeroed space not reclaimed on downsize
4. ⚠️ **Permissionless ExtendProgram**: Can be griefed
5. ⚠️ **Authority Synchronization**: Buffer and program authorities must match

**Status**: Fixes deferred to future phases or LoaderV4 migration.

### LoaderV4 Considerations

- ⚠️ Not yet fully integrated (builtin → on-chain transition planned)
- ⚠️ Migration tooling may be incomplete
- ⚠️ No forced migration timeline announced yet
- ⚠️ May have different performance characteristics (monitor benchmarks)

---

## 10. Testing Checklist

Before deploying to Solana 3.x, verify:

- [ ] Program deployment with `UpgradeableLoaderState::ProgramData` works
- [ ] ProgramData derivation via `find_program_address()` succeeds
- [ ] ELF validation passes during deployment
- [ ] Program upgrades work correctly
- [ ] Authority transfers function properly
- [ ] Rent calculations are correct
- [ ] Program cache updates on deployment
- [ ] Executable flag set correctly on program accounts

---

## 11. Resources & References

### Official Documentation
- [Solana Programs Docs](https://solana.com/docs/core/programs)
- [Deploying Programs Guide](https://solana.com/docs/programs/deploying)
- [RFC-0423: Loader V3: Fix or Replace](https://github.com/solana-foundation/solana-improvement-documents/discussions/423)

### API Documentation
- [solana_loader_v3_interface (6.x+)](https://docs.rs/solana-loader-v3-interface/)
- [solana_loader_v4_interface (3.x+)](https://docs.rs/solana-loader-v4-interface/)
- [solana-program bpf_loader_upgradeable](https://docs.rs/solana-program/latest/solana_program/bpf_loader_upgradeable/)

### MagicBlock References
- [loader.rs](file:///Users/thlorenz/dev/mb/wizard/magicblock-validator/magicblock-processor/src/loader.rs)
- [bpf_loader_v1.rs](file:///Users/thlorenz/dev/mb/wizard/magicblock-validator/magicblock-account-cloner/src/bpf_loader_v1.rs)

---

## 12. Summary Table: API Stability

| Feature | Solana 2.x | Solana 3.x | Status |
|---------|-----------|-----------|--------|
| `UpgradeableLoaderState` enum | ✅ Present | ✅ Present | Stable |
| `Program` variant | ✅ Yes | ✅ Yes | Stable |
| `ProgramData` variant | ✅ Yes | ✅ Yes | Stable |
| `Buffer` variant | ✅ Yes | ✅ Yes | Stable |
| `bpf_loader_upgradeable::id()` | ✅ Yes | ✅ Yes | Stable |
| `find_program_address()` | ✅ Yes | ✅ Yes | Stable |
| `Migrate` instruction | ❌ No | ✅ New | Optional |
| LoaderV4 | ❌ No | ✅ New | Builtin (optional) |
| LoaderV4 as on-chain | N/A | ⏳ Planned | Future |

---

## Conclusion

**MagicBlock's loader.rs code is fully compatible with Solana 3.x without any modifications.** The BPF loader interfaces remain stable, and no breaking changes have been introduced to LoaderV3. LoaderV4 is available as an optional alternative, with migration being opt-in at this time.

No action is required for Solana 3.x compatibility, but monitoring the LoaderV4 roadmap is recommended for future planning.

---

**Last Updated**: January 15, 2026  
**Verified Against**: 
- Solana 3.0+ documentation
- solana-loader-v3-interface 6.x+
- solana-loader-v4-interface 3.x
- RFC-0423 latest consensus
