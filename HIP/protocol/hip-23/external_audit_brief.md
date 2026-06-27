# HIP-23 External Audit Brief

Version: 1.0  
Date: 2026-06-22  
Branch: `hip-23-draft`  
Repository: `hacash/fullnodedev` (fork: `Moskyera/fullnodedev`)

---

## 1. Engagement summary

| Item | Value |
|------|-------|
| Type | Application-layer integration standard (no consensus fork) |
| Scope | Patterns P1–P5, wallet/indexer guidance, test harness |
| Out of scope | Consensus, HVM v2, HIP-25, mempool policy beyond sig/duplicate |

---

## 2. Artifacts for auditors

| Document | Path |
|----------|------|
| Normative spec | `spec.md` |
| JSON templates | `templates.md` |
| Threat model | `threat_model.md` |
| Invariants | `invariants.md` |
| Traceability (26/26 MUST) | `requirements_traceability.md` |
| Findings register | `audit_findings.md` |
| Wallet checklist | `wallet_checklist.md` |
| Indexer dictionary | `indexer_dictionary.md` |
| Test vectors | `[hip23-integration/tests/fixtures/hip23_test_vectors.json](https://github.com/Moskyera/hip23-integration)` |
| Error classifier | `[hip23-integration/tests/common/hip23_errors.rs](https://github.com/Moskyera/hip23-integration)` |

---

## 3. Verification commands

```bash
# Full HIP-23 suite (~85+ named tests + proptest)
cargo test hip23_ -- --nocapture

# Workspace regression
cargo test --workspace

# Optional libfuzzer (requires cargo-fuzz)
cd fuzz && cargo fuzz run tex_cell_act_parse -- -max_total_time=30
```

---

## 4. Internal sign-off (pre-external)

- [x] MUST requirements traced to tests
- [x] Production path (`fast_sync=false`) covered
- [x] Chain fork/merge + duplicate-tx on chain path
- [x] TEX replay wire + persisted chain demonstration
- [x] Stable error code classifier (F-007 mitigation)
- [x] Cross-implementation test vector registry
- [x] Proptest + fuzz-adjacent parse property
- [ ] External auditor report (pending engagement)

---

## 5. Residual accepted risks

| Risk | Mitigation |
|------|------------|
| TEX replay across txs | Wallet MUST pin full composed tx (`HIP23_wallet_checklist.md`) |
| Guard errors are strings on-chain | Indexer uses `hip23_errors.rs` classifier |
| P4 non-atomic two-tx flow | Documented; indexers correlate by serial+issuer |

---

## 6. Contact / process

Report security issues per `SECURITY.md`. HIP-23 changes require `cargo test hip23_` pass before merge.