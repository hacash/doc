# HIP-23 Threat Model

Version: 1.0  
Date: 2026-06-22  
Audience: security reviewers, wallet/indexer engineers, integrators  
Baseline: `spec.md` v1.1 draft

---

## 1. Scope

This document models threats to **integrators** building on HIP-23 patterns P1–P5. It does not change consensus rules. Assumptions:

- Istanbul is active (`height >= 765432` on mainnet).
- Full nodes run with `fast_sync = false` in production (signature + duplicate-tx checks enabled).
- HVM companion contracts are out of scope (HIP-23 v1).

---

## 2. Assets & trust boundaries

| Asset | Owner | Threat if lost |
|-------|-------|----------------|
| Private keys (main, TEX signers) | User / counterparty | Direct fund loss |
| Signed TEX bundles | Counterparty | Replay in unintended composed tx |
| Composed Type3 tx draft | Coordinator | Wrong pairing, ordering, gas |
| Indexer classification | Service operator | Wrong UX, compliance, accounting |
| Block inclusion timing | Miner / network | Height-guard expiry (P2) |

**Trust boundaries:**

```
Wallet A  <--co-sign TEX-->  Wallet B
     |                              |
     v                              v
Coordinator (optional) ----> Full node (consensus)
     |
     v
Indexer / explorer (read-only, MUST NOT authorize spends)
```

Users MUST NOT trust indexers or coordinators for authorization — only validated composed txs and signatures.

---

## 3. STRIDE analysis

| Category | Threat | Patterns | Mitigation |
|----------|--------|----------|------------|
| **Spoofing** | Counterparty TEX `addr` mismatch | P1, P4 | Wallet verifies `TexCellAct.addr` before co-sign |
| **Tampering** | Cells altered after TEX sign | P1, P4 | Signature over `addr+cells`; strict-path tests |
| **Tampering** | Main Type3 signature tamper | All | `fast_sync=false` rejects (`hip23_production_tampered_main_signature_rejected`) |
| **Repudiation** | “I didn’t agree to this swap” | P1 | Pin full composed tx hash before co-sign; store quote |
| **Information disclosure** | Leaked co-sign bundle | P1 | Bundle alone cannot move funds without inclusion in valid tx |
| **Denial of service** | Under-gassed AST/TEX | P4, P5 | Pre-validate `gas_max`; show gas estimate |
| **Denial of service** | Height window miss | P2 | Show inclusive `[start,end]`; simulate at tip |
| **Elevation** | Guard-only tx topology | P2, P3 | Precheck rejects (`hip23_topology_guard_only_*`) |
| **Elevation** | `AssetCreate` + TEX same tx | P4 | `TOP_ONLY` rejected (`hip23_p4_asset_create_with_tex_same_tx_rejected`) |

---

## 4. Pattern-specific threats

### P1 — Atomic TEX swap

| ID | Threat | Severity | On-chain result | Off-chain MUST |
|----|--------|----------|-----------------|----------------|
| T-P1-1 | Imbalanced pay/get | High | Settlement fault | Zero-sum precheck |
| T-P1-2 | TEX replay in different composed tx | Medium | Both txs may settle if funded | Pin full tx hash |
| T-P1-3 | Post-sign cell tamper | High | Sig verify fail | Re-verify before broadcast |
| T-P1-4 | Asset cells without gas | Medium | `gas not initialized` | `gas_max > 0` |

### P2 — Height-guarded payment

| ID | Threat | Severity | On-chain result | Off-chain MUST |
|----|--------|----------|-----------------|----------------|
| T-P2-1 | Debit before guard (simulator confusion) | Medium | Whole tx reverts | Atomic simulation |
| T-P2-2 | Inclusive boundary off-by-one | Low | Revert outside window | Test `start` and `end` |
| T-P2-3 | Using P5 when P2 semantics needed | Medium | Else branch pays | Pattern selection review |

### P3 — BalanceFloor

| ID | Threat | Severity | On-chain result | Off-chain MUST |
|----|--------|----------|-----------------|----------------|
| T-P3-1 | Floor before debit | High | Protection bypassed | Enforce debit→floor order |
| T-P3-2 | Floor ignores post-fee balance | Medium | Floor passes, fee drains later | Model fee in floor value |
| T-P3-3 | Zero dimension not guarded | Low | Unprotected asset/HAC | Set non-zero fields |

### P4 — HIP20 + TEX

| ID | Threat | Severity | On-chain result | Off-chain MUST |
|----|--------|----------|-----------------|----------------|
| T-P4-1 | TEX before asset exists | High | Fault | Tx B after Tx A |
| T-P4-2 | Wrong `protocol_cost` | High | Fault | Quote `block_reward(height)` |
| T-P4-3 | Issuer not signing pay cells | Medium | Settlement fail | Issuer co-sign |
| T-P4-4 | Serial below minsri | High | Fault | Height-aware serial |

### P5 — AST conditional

| ID | Threat | Severity | On-chain result | Off-chain MUST |
|----|--------|----------|-----------------|----------------|
| T-P5-1 | Condition fault vs revert confusion | Medium | Abort vs else | UX labels (`cond_outcome`) |
| T-P5-2 | Zero gas | Medium | Fault | `gas_max > 0` |
| T-P5-3 | Signatures for both branches required | Low | Broadcast fail | Collect all branch signers |

---

## 5. Attacker profiles

| Profile | Capability | Primary risk |
|---------|------------|--------------|
| Malicious counterparty | Co-signs TEX, changes composed tx | T-P1-2, T-P1-3 |
| Malicious coordinator | Builds final Type3 | Ordering, extra actions |
| Network observer | Read mempool | Front-running N/A for atomic TEX in same tx |
| Rogue indexer | Wrong labels | Accounting / compliance |
| Miner | Censor or delay | P2 expiry |

---

## 6. Residual risks (accepted in v1)

1. **TEX replay:** Signatures are not bound to a specific composed tx hash — wallets MUST pin full tx (see `tests/hip23_tex_replay.rs`).
2. **`fast_sync` test harness:** Most HIP-23 tests skip sig/duplicate checks; production-path suite required before release.
3. **No stable guard reason codes:** Indexers parse error strings (see `HIP23_indexer_dictionary.md`).
4. **P4 non-atomic:** Issuance and distribution are two txs — integrators MUST handle partial completion.

---

## 7. Verification mapping

| Control | Test suite |
|---------|------------|
| Pattern semantics | `hip23_pattern_{regression,adversarial,stress}.rs` |
| Production policy | `hip23_production_path.rs`, `hip23_audit_strict.rs` |
| Chain atomicity | `hip23_chain_integration.rs` |
| TEX replay awareness | `hip23_tex_replay.rs` |
| Generative properties | `hip23_proptest.rs` |

Run before release:

```bash
cargo test hip23_ -- --nocapture
```