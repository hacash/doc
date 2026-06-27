# HIP-23 Audit Findings Register

Version: 1.0  
Date: 2026-06-22  
Method: internal audit per `HIP23_audit_scope.md`

---

## Summary

| Severity | Open | Fixed | Accepted |
|----------|------|-------|----------|
| High | 0 | 2 | 1 |
| Medium | 0 | 3 | 2 |
| Low | 0 | 3 | 0 |
| Informational | 2 | 0 | 3 |

No critical consensus issues (expected — HIP-23 is application-layer).

---

## Findings

### F-001 — TEX signatures replayable across txs

| Field | Value |
|-------|-------|
| Severity | **Accepted / High (integrator)** |
| Pattern | P1, P4 |
| Status | Documented + tested |

**Description:** `TexCellAct` signs `addr + cells` only. The same valid bundle may appear in multiple composed Type3 txs if counterparties co-sign without pinning the full tx.

**Proof:** `hip23_tex_replay_same_bundle_different_main_succeeds` (wire replay via `clone_tex_wire`)

**Remediation:** Wallet checklist §co-signing; `HIP23_threat_model.md` T-P1-2. Not a protocol bug.

---

### F-002 — P3 floor-before-debit bypasses protection

| Field | Value |
|-------|-------|
| Severity | **High** |
| Pattern | P3 |
| Status | **Fixed** (documented + tested) |

**Description:** `BalanceFloor` reads in-tx state at guard index. Floor listed before debit checks pre-debit balance.

**Proof:** `hip23_p3_floor_before_transfer_checks_pre_debit_state`

**Remediation:** `HIP23_wallet_checklist.md` P3 ordering MUST; `spec.md` §6.4.

---

### F-003 — P2 debit-before-guard confuses simulators

| Field | Value |
|-------|-------|
| Severity | **Medium** |
| Pattern | P2 |
| Status | **Fixed** (documented + tested) |

**Description:** Wrong ordering still fails whole tx on-chain, but step simulators may show debit before revert.

**Proof:** `hip23_p2_transfer_before_guard_still_reverts_outside_window`, `hip23_chain_failed_tx_does_not_commit`

**Remediation:** `spec.md` §5.4; wallet atomic simulation requirement.

---

### F-004 — P4 AssetCreate + TEX same tx rejected

| Field | Value |
|-------|-------|
| Severity | **Medium** |
| Pattern | P4 |
| Status | **Fixed** (by design) |

**Description:** `AssetCreate` is `TOP_ONLY`; combined tx fails topology check.

**Proof:** `hip23_p4_asset_create_with_tex_same_tx_rejected`

**Remediation:** Two-tx flow documented in §7.

---

### F-005 — P5 condition fault vs revert

| Field | Value |
|-------|-------|
| Severity | **Medium** |
| Pattern | P5 |
| Status | **Fixed** (documented) |

**Description:** Invalid height range faults entire `AstIf`; outside window reverts to else. Wallets conflating these mis-label tx outcome.

**Proof:** `hip23_p5_ast_if_condition_fault_aborts_whole_node`, `hip23_p5_ast_else_branch_executes_transfer`

**Remediation:** `HIP23_indexer_dictionary.md` `cond_outcome` + `ast_branch`.

---

### F-006 — fast_sync test harness skips production checks

| Field | Value |
|-------|-------|
| Severity | **Medium (process)** |
| Pattern | All |
| Status | **Mitigated** |

**Description:** ~85% of HIP-23 tests use `fast_sync=true`, skipping signature and duplicate-tx validation.

**Proof:** `[hip23-integration/tests/common/hip23.rs](https://github.com/Moskyera/hip23-integration)`; production suites added.

**Remediation:** `hip23_production_path.rs`, `hip23_audit_strict.rs`, `SECURITY.md` release gate.

---

### F-007 — No stable guard reason-code enum

| Field | Value |
|-------|-------|
| Severity | **Low** |
| Pattern | P2, P3, P5 |
| Status | **Mitigated** (v1 classifier) |

**Description:** On-chain errors remain strings; no consensus enum in v1.

**Remediation:** `[hip23-integration/tests/common/hip23_errors.rs](https://github.com/Moskyera/hip23-integration)` + `hip23_guard_error_codes.rs` lock stable indexer codes; `HIP23_indexer_dictionary.md` §3.

---

### F-008 — Proptest setup guard drop (fixed)

| Field | Value |
|-------|-------|
| Severity | **Low** |
| Pattern | Test infra |
| Status | **Fixed** |

**Description:** Repeated `enable_mint_setup()` in proptest cases cleared scoped registry.

**Proof:** `ensure_standard_protocol_setup_for_tests` in `init_setup()`.

---

### F-009 — Main signature tamper rejected in production path

| Field | Value |
|-------|-------|
| Severity | Informational |
| Pattern | All |
| Status | Verified |

**Proof:** `hip23_production_tampered_main_signature_rejected`, `hip23_audit_strict_main_sig_tamper_rejected`

---

### F-010 — Imbalanced TEX always fails settlement

| Field | Value |
|-------|-------|
| Severity | Informational |
| Pattern | P1 |
| Status | Verified |

**Proof:** `hip23_p1_tex_imbalanced_*`, proptest `hip23_proptest_imbalanced_tex_always_fails`

---

## Recommendations (informational)

| ID | Recommendation | Priority |
|----|----------------|----------|
| R-01 | External third-party audit before mainnet wallet launch | High |
| R-02 | Add `cargo-fuzz` on TEX parse | Medium (**done**: `fuzz/` + proptest parse property) |
| R-03 | CI gate on `cargo test hip23_` | Medium (**done**: `.github/workflows/hip23.yml`) |
| R-04 | Expand proptest to P4/P5 properties | Low (**done**) |
| R-05 | Cross-implementation test vectors file | Low (**done**: `[hip23-integration/tests/fixtures/hip23_test_vectors.json](https://github.com/Moskyera/hip23-integration)`) |

---

## Sign-off criteria (internal)

- [x] All High findings documented with wallet mitigations
- [x] Production-path tests pass
- [x] Chain fork semantics tested
- [x] Traceability matrix ≥ 90% MUST coverage
- [ ] External auditor review (pending)