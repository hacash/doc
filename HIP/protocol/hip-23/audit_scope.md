# HIP-23 Audit Scope Document

Version: 1.0  
Date: 2026-06-22  
Status: Internal / pre-external audit  
Branch: `hip-23-draft`

---

## 1. Engagement type

Application-layer integration standard audit — **no consensus fork**. Validates that documented patterns P1–P5 match `fullnodedev` behavior and that wallet/indexer guidance is testable.

---

## 2. In scope

| Component | Path | Notes |
|-----------|------|-------|
| HIP-23 specification | `spec.md` | Normative MUST/SHOULD |
| JSON templates | `templates.md` | Structural validity |
| TEX settlement | `protocol/src/tex/*` | Zero-sum, signatures |
| Guards | `protocol/src/action/chain.rs` | HeightScope, BalanceFloor, ChainAllow |
| AST conditionals | `protocol/src/action/ast*` | P5 branch selection |
| HIP20 AssetCreate | `mint/src/action/asset.rs` | P4 issuance rules |
| Type3 execute path | `protocol/src/transaction/type3.rs` | Settlement + gas |
| Chain tx isolation | `chain/src/check.rs` | Fork/merge semantics |
| HIP-23 test suite | `tests/hip23_*.rs` | All layers |
| Audit artifacts | `doc/HIP23_*.md`, `SECURITY.md` | This package |

---

## 3. Out of scope

- Consensus / fork activation mechanics
- HVM contract templates (HIP-23 v2)
- Cross-chain bridges, oracles, MEV
- Mempool policy beyond duplicate-tx + signature (node-specific)
- UI/UX of third-party wallets
- Economic modeling of `protocol_cost` / fees
- HIP-25 (separate track)

---

## 4. Assumptions

1. Istanbul active at `ONLINE_OPEN_HEIGHT = 765432`.
2. Production nodes use `fast_sync = false`.
3. Integrators read `HIP23_wallet_checklist.md` before co-signing.
4. Test height baseline: `TEST_HEIGHT = 775432` in harness.

---

## 5. Methodology (aligned with industry practice)

| Phase | Activity | Artifact |
|-------|----------|----------|
| 1. Spec review | MUST/SHOULD extraction | `HIP23_requirements_traceability.md` |
| 2. Threat modeling | STRIDE per pattern | `HIP23_threat_model.md` |
| 3. Invariant catalog | Formal properties | `HIP23_invariants.md` |
| 4. Test review | Map tests → requirements | §11 `spec.md` + traceability |
| 5. Adversarial testing | Negative paths | `hip23_pattern_adversarial.rs` |
| 6. Production path | Strict-mode smoke | `hip23_production_path.rs`, `hip23_audit_strict.rs` |
| 7. Property testing | Generative cases | `hip23_proptest.rs` |
| 8. Chain semantics | Fork/merge atomicity | `hip23_chain_integration.rs` |
| 9. Replay analysis | TEX signature scope | `hip23_tex_replay.rs` |
| 10. Findings log | Severity + remediation | `HIP23_audit_findings.md` |

---

## 6. Severity taxonomy

| Level | Definition |
|-------|------------|
| Critical | Fund loss or consensus divergence (N/A for HIP-23 — app layer) |
| High | Integrator fund loss with normal wallet |
| Medium | Failed tx / UX / accounting error |
| Low | Documentation or non-exploitable inconsistency |
| Informational | Hardening recommendation |

---

## 7. Deliverables checklist

- [x] Threat model
- [x] Invariant catalog
- [x] Requirements traceability matrix
- [x] Wallet checklist (v1.1)
- [x] Indexer dictionary (v1.1)
- [x] Findings register
- [x] Chain + replay integration tests
- [x] Strict-path adversarial mirror
- [x] `SECURITY.md`
- [x] External audit brief (`HIP23_external_audit_brief.md`)
- [x] Stable error classifier (`[hip23-integration/tests/common/hip23_errors.rs](https://github.com/Moskyera/hip23-integration)`)
- [x] Cross-implementation test vectors (`[hip23-integration/tests/fixtures/hip23_test_vectors.json](https://github.com/Moskyera/hip23-integration)`)
- [x] `cargo-fuzz` target (`fuzz/fuzz_targets/tex_cell_act_parse.rs`)
- [ ] External third-party audit engagement (process only)

---

## 8. Test commands for auditors

```bash
cargo test hip23_ -- --nocapture
cargo test hip23_audit_ -- --nocapture
cargo test hip23_chain_ -- --nocapture
cargo test hip23_tex_replay_ -- --nocapture
```