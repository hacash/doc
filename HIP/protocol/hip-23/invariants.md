# HIP-23 Protocol Invariants

Version: 1.0  
Date: 2026-06-22  
Audience: auditors, protocol developers, QA  
Model: `vm/doc/runtime-spec.md` §11, `doc/diamond-insc.md` §4.2

---

## 1. Global invariants (all patterns)

| ID | Invariant | Enforcement |
|----|-----------|-------------|
| G-1 | Type3 txs with Istanbul actions use `type >= 3` | `check_gated_*` |
| G-2 | Guard-only topologies rejected at precheck | `precheck_tx_actions` |
| G-3 | `do_settlement()` runs after top-level actions on Type3 | `type3.rs` execute path |
| G-4 | Failed tx execution does not commit partial state (chain path) | `chain/src/check.rs` fork/merge |
| G-5 | Each `TexCellAct` signature covers `addr + cells` only | TEX sign/verify |
| G-6 | Mainnet Istanbul height `>= 765432` unless dev bypass | `upgrade.rs` |

---

## 2. TEX settlement invariants (P1, P4 Tx B)

| ID | Invariant | Violation symptom |
|----|-----------|-------------------|
| T-1 | Σ zhu pay = Σ zhu get per tx | `coin settlement check failed` |
| T-2 | Σ sat pay = Σ sat get per tx | `coin settlement check failed` |
| T-3 | Σ dia pay = Σ dia get per tx | `diamonds settlement check failed` |
| T-4 | Σ asset(serial) pay = Σ asset(serial) get per tx | `asset <n> settlement check failed` |
| T-5 | TEX cells execute before settlement | Ledger updated pre-`do_settlement()` |
| T-6 | Asset TEX cells (`7/8`) require `gas_max > 0` | `gas not initialized` |

**Zero-sum property (formal):** For each dimension `d ∈ {zhu, sat, dia, assets*}`,  
`Σ debits_d = Σ credits_d` at settlement boundary.

---

## 3. Guard invariants (P2, P3, P5 cond)

| ID | Invariant | Notes |
|----|-----------|-------|
| GU-1 | Guards observe state after all lower-index actions | Sequential action index |
| GU-2 | `HeightScope` revert is inclusive `[start, end]` | `end=0` → unbounded above |
| GU-3 | `HeightScope` invalid range (`start > end`, `end≠0`) → **fault** | Not revert |
| GU-4 | `BalanceFloor` checks in-tx balance at guard position | Not pre-tx chain state |
| GU-5 | Guard revert → action error unwind (tx fails for top-level guards) | P2 whole-tx fail |
| GU-6 | P5 cond revert → `br_else`; cond fault → abort `AstIf` | See `doc/ast-spec.md` |

---

## 4. Pattern-specific invariants

### P2

- **P2-1:** `HeightScope` index < debit action index (MUST).
- **P2-2:** Outside window → tx fails entirely (no partial commit on chain).

### P3

- **P3-1:** Debit index < `BalanceFloor` index (MUST).
- **P3-2:** At least one floor field non-zero (MUST).
- **P3-3:** Type3 fee debited after all actions — floor does not include fee unless modeled.

### P4

- **P4-1:** `AssetCreate` is sole top action in Tx A (`TOP_ONLY`).
- **P4-2:** `protocol_cost == genesis::block_reward(height)` exactly.
- **P4-3:** Asset serial ≥ minsri at height on mainnet.
- **P4-4:** Tx B TEX only after asset exists in state (same block order or later).

### P5

- **P5-1:** `gas_max > 0` when AST depth > 0.
- **P5-2:** Cond `AstSelect` uses `exe_min = exe_max = 1` for single predicates.
- **P5-3:** Gas consumed on cond attempt is not refunded on revert.

---

## 5. Chain execution invariants

| ID | Invariant | Test |
|----|-----------|------|
| C-1 | `try_execute_tx_by` forks from accumulated block state | `hip23_chain_*` |
| C-2 | Success → `merge_sub`; failure → discard fork | `hip23_chain_failed_tx_does_not_commit` |
| C-3 | Sequential successful txs compose (P4 A→B) | `hip23_chain_p4_tx_a_then_b_commits` |

---

## 6. Known non-invariants (documented exceptions)

1. **TEX signatures are replayable** across txs with identical cells — not a bug; wallets must pin composed tx.
2. **`fast_sync=true`** disables signature and duplicate-tx checks — test-only relaxation.
3. **In-tx partial progress** visible in direct `tx.execute()` simulators — not chain-persistent on failure.

---

## 7. Regression proof

Each invariant ID maps to tests in `requirements_traceability.md`. Run:

```bash
cargo test hip23_ -- --nocapture
```