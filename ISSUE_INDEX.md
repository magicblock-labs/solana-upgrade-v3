# Issue Index: Solana 2.x → 3.x Upgrade Findings

**Quick reference for all 21 investigated issues**  
**Status**: ✅ ALL VERIFIED

---

## Issues by Category

### Account Types & Operations (Issues 1.1-1.4)

| # | Issue | Finding | Impact | Effort | Status |
|---|-------|---------|--------|--------|--------|
| **1.1** | AccountSharedData Enum Shape | Custom fork maintains Owned/Borrowed; official 3.x eliminates | Fork rebase required | 4-5 weeks | 🔴 BLOCKER |
| **1.2** | AccountSeqLock API | 100% MagicBlock custom (not in official Solana) | Must maintain in fork | 4-5 weeks | 🔴 BLOCKER |
| **1.3** | ReadableAccount/WritableAccount | Official 3.x traits stable; no breaking changes | None to official traits | 0 hours | ✅ GREEN |
| **1.4** | Fork-Specific Account Flags | 9 custom methods; bit field storage | Core architecture; cannot remove | 4-5 weeks | 🔴 BLOCKER |

**Summary**: Fork rebase mandatory; official traits compatible.

---

### Transaction Processing APIs (Issues 2.1-2.5)

| # | Issue | Finding | Breaking? | Effort | Status |
|---|-------|---------|-----------|--------|--------|
| **2.1** | TransactionProcessingEnvironment Fields | 5 field changes (removed, type change, added) | ✅ YES | 5-9 hrs | 🔴 BREAK |
| **2.2** | load_and_execute_sanitized_transactions | CheckedTransactionDetails constructor parameter type change | ✅ YES | 1-2 hrs | 🔴 BREAK |
| **2.3** | ProcessedTransaction Enum | Account storage: Vec<tuple> → Vec<KeyedAccount> | ✅ YES | 1.5-2 hrs | 🔴 BREAK |
| **2.4** | AccountsBalances Type | No changes (pre/post still Vec<u64>) | ❌ NO | 0 hours | ✅ GREEN |
| **2.5** | Fee-Only Semantics | Stable; implementation correct | ❌ NO | 0 hours | ✅ GREEN |

**Summary**: 3 breaking changes; 2 stable. Total refactoring: 3-5 hours.

---

### Program Loading & Builtins (Issues 3.1-3.3)

| # | Issue | Finding | Action | Effort | Status |
|---|-------|---------|--------|--------|--------|
| **3.1** | Crate Version Skew | solana-loader-v3 (3.0) conflicts with solana-* (2.2) | Resolve version alignment | BLOCKER | 🔴 CRITICAL |
| **3.2** | BuiltinFunctionWithContext | Type stable; InvokeContext structure changes | Refactor context access | 1-2 days | 🟠 MEDIUM |
| **3.3** | LoaderV3/BPF Interfaces | Fully stable in 3.x (LoaderV4 available, optional) | No changes needed | 0 hours | ✅ GREEN |

**Summary**: Version skew critical blocker; loaders stable.

---

### Core Types (Issues 4.1-4.2)

| # | Issue | Finding | Breaking? | Effort | Status |
|---|-------|---------|-----------|--------|--------|
| **4.1** | Pubkey/Signature Import Paths | All paths work in 2.x and 3.x | ❌ NO | 30 min | ✅ GREEN |
| **4.2** | Instruction/CompiledInstruction | Pubkey→Address; module reorganization; error type rename | ✅ YES | 1-2 weeks | 🔴 BREAK |

**Summary**: Paths stable; Instruction types require 1-2 week migration.

---

### SVM Integration & Forks (Issues 5.1-5.3)

| # | Issue | Finding | Impact | Effort | Status |
|---|-------|---------|--------|--------|--------|
| **5.1** | Fork Rebase Requirements | magicblock-svm (3-4 wks), solana-account (4-5 wks) | MUST complete before upgrade | 7-9 weeks | 🔴 BLOCKER |
| **5.2** | SVMMessage/SanitizedTransaction | get_loaded_addresses() removed; new pattern in 3.x | Boundary refactoring | 2-3 weeks | 🔴 BREAK |
| **5.3** | Account Loader/Rollback APIs | CheckedTransactionDetails, RollbackAccounts, ProcessedTransaction changes | Consolidated from 2.2, 2.3, 2.5 | 3-5 hrs | 🔴 BREAK |

**Summary**: Fork rebases critical blocker (7-9 weeks); SanitizedTransaction boundary redesign (2-3 weeks).

---

### Feature Sets & Gates (Issues 6.1-6.2)

| # | Issue | Finding | Breaking? | Effort | Status |
|---|-------|---------|-----------|--------|--------|
| **6.1** | FeatureSet Structure | `active` field private; use getter method | ✅ YES | 10 min | 🟢 MINOR |
| **6.2** | Feature Account Persistence | SVM 3.x ignores on-chain accounts; uses FeatureSet only | Architectural mismatch | 1 hour | 🔴 SEMANTIC |

**Summary**: Minor API change (6.1); architectural shift in feature semantics (6.2).

---

## Issues by Status

### 🔴 Critical Blockers (Must Decide/Fix First)
1. **5.1** Fork Rebase Requirements — 7-9 weeks
2. **3.1** Crate Version Skew — Decide strategy
3. **6.2** Feature Account Semantics — Architectural change
4. **1.1-1.4** Account Custom Extensions — Fork rebase includes this

### 🔴 Breaking Changes (Require Code Updates)
1. **2.1** TransactionProcessingEnvironment — 5-9 hours
2. **2.2** CheckedTransactionDetails — 1-2 hours
3. **2.3** ProcessedTransaction — 1.5-2 hours
4. **4.2** Instruction Types — 1-2 weeks
5. **5.2** SanitizedTransaction Boundary — 2-3 weeks
6. **3.2** Builtin Context — 1-2 days
7. **6.1** FeatureSet API — 10 minutes

### 🟠 Medium Impact
1. **3.2** Builtin Context Changes — 1-2 days

### ✅ Green Light (No Changes Needed)
1. **1.3** ReadableAccount/WritableAccount
2. **2.4** AccountsBalances
3. **2.5** Fee-Only Semantics
4. **3.3** Loader Interfaces
5. **4.1** Import Paths

---

## Issues by Effort

### Very Quick (< 1 hour)
- 6.1 FeatureSet API — 10 min
- 6.2 Feature Persistence — 1 hour
- 4.1 Import Paths — 30 min
- 2.2 CheckedTransactionDetails — 1-2 hours
- 2.3 ProcessedTransaction — 1.5-2 hours

### Medium (1 day - 1 week)
- 2.1 TransactionProcessingEnvironment — 5-9 hours
- 5.3 Account Loader/Rollback APIs — 3-5 hours
- 3.2 Builtin Context — 1-2 days
- 4.2 Instruction Types — 1-2 weeks

### Long Term (1+ weeks)
- 5.2 SanitizedTransaction — 2-3 weeks
- 5.1 Fork Rebases — 7-9 weeks

---

## Critical Path

**Must complete in order**:

1. **DECIDE fork strategy** (Issue 5.1) — Sets timeline for everything
2. **RESOLVE version skew** (Issue 3.1) — Required for compilation
3. **REBASE forks** (Issues 1.1-1.4, 5.1) — 7-9 weeks parallel work
4. **UPDATE transaction processing** (Issues 2.1-2.3) — 3-5 hours after fork rebase
5. **MIGRATE instruction types** (Issue 4.2) — 1-2 weeks, can start once forks mergeable
6. **ADAPT SanitizedTransaction** (Issue 5.2) — 2-3 weeks, depends on fork rebase
7. **UPDATE feature semantics** (Issue 6.2) — 1 hour once strategy decided

---

## Risk Matrix

```
                   IMPACT
         Low      Medium     High
       ┌─────────────────────────────┐
    L  │ 4.1      │ 3.2    │ 6.1    │
       │          │        │        │
RISK M │ 2.4      │ 2.2    │ 2.1    │
       │ 2.5      │ 2.3    │        │
       │          │ 3.3    │        │
    H  │          │        │ 5.1    │
       │          │ 5.2    │ 5.3    │
       │          │ 6.2    │        │
       │          │ 4.2    │ 1.1-4  │
       │          │        │ 3.1    │
       └─────────────────────────────┘
```

**High Risk + High Impact** (Top Priority):
- 5.1 Fork Rebase
- 1.1-1.4 Account Custom Extensions
- 3.1 Version Skew

**Medium Risk + Medium Impact**:
- 4.2 Instruction Types
- 5.2 SanitizedTransaction
- 2.1-2.3 Transaction APIs

---

## File Mapping

| Issue # | File | Lines | Impact |
|---------|------|-------|--------|
| 1.1-1.4 | magicblock-core/src/link/accounts.rs | 40-114 | Account enum, seqlock, flags |
| 2.1 | magicblock-processor/src/lib.rs | 15-53 | SVM environment construction |
| 2.2 | magicblock-processor/src/executor/processing.rs | 126-129 | Constructor call |
| 2.3 | magicblock-processor/src/executor/processing.rs | 150-160, 292-302 | Pattern match, field access |
| 2.5 | magicblock-processor/src/executor/processing.rs | 54-61, 273-305 | Fee-only handling |
| 3.2 | magicblock-processor/src/executor/mod.rs | 107-120 | Builtin registration |
| 3.3 | magicblock-processor/src/loader.rs | 15-71 | Program loading |
| 4.2 | Multiple (Instruction usage throughout) | - | Widespread impact |
| 5.1 | Cargo.toml | workspace.dependencies | Fork pinning |
| 5.2 | magicblock-processor/src/executor/processing.rs | 195, 209 | Ledger meta assembly |
| 6.1 | magicblock-processor/src/lib.rs | 37 | FeatureSet::active access |
| 6.2 | magicblock-processor/src/lib.rs | 35-40 | Feature account creation |

---

## Decision Matrix

### Decision 1: Fork Strategy
| Option | Timeline | Effort | Risk |
|--------|----------|--------|------|
| A: Rebase forks | Now | 7-9 weeks | Medium-High |
| B: Contribute upstream | 6+ months | Unknown | Low |
| C: Redesign (no forks) | 2-3 months | 10+ weeks | Very High |

**Recommendation**: Option A

### Decision 2: Version Alignment
| Option | Timeline | Effort | Impact |
|--------|----------|--------|--------|
| A: Stay 2.2.x | Now | 0 hours | Defer upgrade 2-3 months |
| B: Full 3.x migration | 11-15 weeks | Full scope | All issues |

**Recommendation**: Option A (then Option B in 2-3 months)

### Decision 3: Feature Account Behavior
| Option | Approach | Consequence |
|--------|----------|-------------|
| A: Keep on-chain + FeatureSet | Dual-track | Validator divergence risk |
| B: FeatureSet only | 3.x compatible | Behavior change; audit needed |

**Recommendation**: Option B (requires stakeholder discussion)

---

## Verification Evidence

All findings based on:
- ✅ Official Solana 3.x crate API inspection
- ✅ MagicBlock fork source code analysis
- ✅ GitHub Agave repository code review
- ✅ Type definition comparison (2.x vs 3.x)
- ✅ Breaking change analysis
- ✅ Impact assessment on MagicBlock codebase

**Zero unverified assumptions** — all "may change" claims from initial analysis have been confirmed or refuted.

---

## How to Use This Index

1. **Find your issue**: Search by # or category
2. **Check status**: Green (no change) or Red (breaking)
3. **Estimate effort**: Use effort column
4. **Find document**: Search `/tmp/solana-upgrade/` directory
5. **Cross-reference**: Use file mapping section

**For details**: Read corresponding detailed issue document (e.g., `1_1-accountshareddata-enum.md`)

---

**Next**: Read [FINDINGS_SUMMARY.md](FINDINGS_SUMMARY.md) for comprehensive analysis
