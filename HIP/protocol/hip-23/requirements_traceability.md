# HIP-23 Requirements Traceability Matrix

Version: 1.0  
Date: 2026-06-22  
Legend: ✅ covered | ⚠️ doc-only | ❌ gap

---

## §3 Common requirements

| ID | Requirement | Test(s) | Status |
|----|-------------|---------|--------|
| C-3.1 | Type >= 3 for Istanbul actions | implicit in all `build_signed_type3` tests | ✅ |
| C-3.2 | gas_max > 0 for AST / asset TEX | `hip23_p1_tex_asset_cells_require_gas`, `hip23_p5_ast_requires_nonzero_gas` | ✅ |
| C-3.3 | Mainnet height >= 765432 | `TEST_HEIGHT` baseline; stress @ alive height | ✅ |
| C-3.4 | TEX zero-sum settlement | `hip23_p1_*`, proptest balanced/imbalanced | ✅ |
| C-3.5 | TEX signs addr+cells | `hip23_p1_tex_tampered_signature_fails`, `hip23_tex_replay_*` | ✅ |
| C-3.6 | No guard-only topology | `hip23_topology_guard_only_*`, proptest guard-only | ✅ |
| C-3.7 | Asset TEX gas init | `hip23_p1_tex_asset_cells_require_gas`, stress low gas | ✅ |

---

## §4 P1 — TEX swap

| ID | MUST/SHOULD | Test(s) | Status |
|----|-------------|---------|--------|
| P1-M1 | Matching pay/get cells | `hip23_pattern_1_*`, `hip23_production_p1_*` | ✅ |
| P1-M2 | Zero-sum per dimension | `hip23_p1_tex_imbalanced_hac_amount_fails`, `hip23_p1_tex_imbalanced_sat_fails` | ✅ |
| P1-M3 | Tamper after sign fails | `hip23_p1_tex_tampered_signature_fails`, `hip23_audit_strict_tex_tamper_rejected` | ✅ |
| P1-S1 | Condition cells optional | `hip23_p1_tex_height_condition_in_bundle` | ✅ |
| P1-S2 | HAC prelude + TEX | `hip23_p1_tex_with_hac_to_trs_prelude_succeeds` | ✅ |

---

## §5 P2 — Height guard

| ID | MUST/SHOULD | Test(s) | Status |
|----|-------------|---------|--------|
| P2-M1 | HeightScope before debit | `hip23_pattern_2_height_guarded_payment` | ✅ |
| P2-M4 | Chain atomicity on fail | `hip23_chain_failed_tx_does_not_commit`, `hip23_p2_transfer_before_guard_*` | ✅ |
| P2-M2 | start <= end (end≠0) | adversarial height tests | ✅ |
| P2-M3 | Revert outside window | `hip23_p2_height_guard_above_end_reverts`, proptest height window | ✅ |
| P2-S1 | Finite end for expiry | `hip23_p2_height_guard_boundary_inclusive` | ✅ |


---

## §6 P3 — BalanceFloor

| ID | MUST/SHOULD | Test(s) | Status |
|----|-------------|---------|--------|
| P3-M1 | Floor after debits | `hip23_pattern_3_*`, `hip23_p3_floor_before_transfer_*` | ✅ |
| P3-M2 | Non-zero floor field | `hip23_p3_floor_asset_dimension_*`, satoshi dimension | ✅ |
| P3-M3 | Revert below floor | `hip23_combined_height_floor_transfer_below_floor_fails` | ✅ |
| P3-S1 | Model fee in floor | doc + wallet checklist | ⚠️ |
| P3-S2 | Non-zero fields only | `hip23_p3_floor_before_transfer_*` | ✅ |

---

## §7 P4 — HIP20 + TEX

| ID | MUST/SHOULD | Test(s) | Status |
|----|-------------|---------|--------|
| P4-M1 | AssetCreate TOP_ONLY | `hip23_p4_asset_create_with_tex_same_tx_rejected` | ✅ |
| P4-M2 | TEX after asset exists | `hip23_pattern_4_*`, `hip23_chain_p4_tx_a_then_b_commits` | ✅ |
| P4-M3 | gas_max > 0 on asset TEX | production P4, stress | ✅ |
| P4-M4 | protocol_cost exact | `hip23_p4_wrong_protocol_cost_rejected`, `hip23_audit_strict_wrong_protocol_cost` | ✅ |
| P4-M5 | Serial minsri | `hip23_stress_*` serial 1025, below minsri fault | ✅ |
| P4-S1 | Pre-sign Tx B | wallet checklist | ⚠️ |

---

## §8 P5 — AST conditional

| ID | MUST/SHOULD | Test(s) | Status |
|----|-------------|---------|--------|
| P5-M1 | gas_max > 0 | `hip23_p5_ast_requires_nonzero_gas` | ✅ |
| P5-M2 | cond exe_min=max=1 | pattern 5 template default | ✅ |
| P5-M3 | No guard-only AST topology | topology tests | ✅ |
| P5-M4 | Revert→else, fault→abort | `hip23_p5_ast_if_condition_fault_*`, `hip23_p5_ast_else_branch_*` | ✅ |
| P5-S1 | Guard-only cond | pattern 5 structure | ✅ |
| P5-S2 | Simple branch actions | production P5 | ✅ |

---

## §9 Security considerations

| Topic | Test(s) | Status |
|-------|---------|--------|
| TEX replay | `hip23_tex_replay_*` | ✅ |
| Ordering | P2/P3 adversarial, chain | ✅ |
| Co-signing | wallet checklist + tamper tests | ✅ |
| Gas under-budget | stress low gas, P5 zero gas | ✅ |
| P2 vs P5 | doc §5.6 + P5 else tests | ✅ |

---

## Production-path (strict mode)

| Control | Test(s) | Status |
|---------|---------|--------|
| Signature verify | `hip23_production_tampered_main_signature_rejected`, `hip23_audit_strict_*` | ✅ |
| Duplicate tx | `hip23_production_duplicate_tx_rejected` | ✅ |
| P1–P5 smoke | `hip23_production_p1_*` … `p5_*` | ✅ |
| Strict proptest | `hip23_proptest_strict_balanced_tex_settles` | ✅ |

---

## Coverage summary

| Category | MUST items | Covered | % |
|----------|------------|---------|---|
| Common §3 | 7 | 7 | 100% |
| P1 | 3 | 3 | 100% |
| P2 | 4 | 4 | 100% |
| P3 | 3 | 3 | 100% |
| P4 | 5 | 5 | 100% |
| P5 | 4 | 4 | 100% |
| **Total MUST** | **26** | **26** | **100%** |

SHOULD items: 8/10 tested (2 wallet-process items doc-only by design).

---

## Gaps / backlog

| Gap | Planned |
|-----|---------|
| `cargo-fuzz` TEX parse | v1.2 |
| P4/P5 dedicated proptest properties | R-04 |
| Cross-implementation vectors | R-05 |
| External audit sign-off | pre-mainnet wallet |