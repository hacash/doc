# HIP-23: Istanbul DeFi Application Patterns

**Status:** v1.1 Ready  
**Type:** Application / integration standard — **no consensus fork**  
**Activation:** Istanbul capabilities @ mainnet height `765432` (`ONLINE_OPEN_HEIGHT`)  
**Depends on:** Type3 transactions, ActionGuard, TEX, AST, HIP20 (`AssetCreate`), HVM (optional)

## 1. Purpose

HIP-23 documents **reusable DeFi composition patterns** on post-Istanbul Hacash. It does not introduce new protocol actions, fork heights, or consensus rules. Wallets, indexers, gateways, and integrators MAY implement these patterns today using existing Istanbul machinery.

Goals:

- Give builders copy-paste-safe transaction templates for common financial flows.
- Tie each pattern to **normative MUST/SHOULD** rules wallets and indexers can validate off-chain.
- Provide regression tests (`[hip23-integration/tests/hip23_pattern_regression.rs](https://github.com/Moskyera/hip23-integration)`) that lock pattern semantics to `fullnodedev`.

Reference material:

- Height gates and Istanbul action registry: `protocol/src/upgrade.rs`
- TEX settlement: `protocol/src/tex/*`, `tests/tex.rs` (`trs1`)
- AST control flow: `doc/ast-spec.md`
- Guards: `protocol/src/action/chain.rs`
- JSON templates: `templates.md`
- Audit package: `threat_model.md`, `invariants.md`, `requirements_traceability.md`, `wallet_checklist.md`, `indexer_dictionary.md`, `audit_scope.md`, `audit_findings.md`, `external_audit_brief.md`
- Error classifier: `[hip23-integration/tests/common/hip23_errors.rs](https://github.com/Moskyera/hip23-integration)`
- Test vectors: `[hip23-integration/tests/fixtures/hip23_test_vectors.json](https://github.com/Moskyera/hip23-integration)`

## 2. Scope

### 2.1 In scope (v1)

| ID | Pattern | Primary primitives |
|----|---------|-------------------|
| P1 | Atomic multi-asset TEX swap | `TexCellAct` (kind 22), transfer cells 1–8 |
| P2 | Time-boxed guarded payment | `HeightScope` (0x0412) + top transfer |
| P3 | BalanceFloor protected transfer | `BalanceFloor` (0x0413) + ordered debits |
| P4 | HIP20 issuance + TEX distribution | `AssetCreate` (16) + asset TEX cells 7/8 |
| P5 | AST conditional settlement | `AstIf` / `AstSelect` + `HeightScope` condition |

### 2.2 Out of scope (v1)

- New action kinds or fork gates
- HVM contract templates (future HIP-23 v2 MAY add optional contract companions)
- Cross-chain bridges, oracle networks, or mempool policy
- Fee-market or MEV recommendations

## 3. Common requirements

All patterns MUST satisfy:

1. **Transaction type:** Use `type >= 3` when submitting `TexCellAct`, AST nodes, or other Istanbul-gated actions that require Type3.
2. **Gas budget:** Set `gas_max > 0` when AST depth > 0 or when asset TEX cells (`cellid` 7 or 8) are present. Plain HAC/SAT/diamond TEX without asset cells MAY use `gas_max = 0` (see P2).
3. **Height gate:** On mainnet (`chain_id = 0`), Istanbul actions MUST only be submitted at `height >= 765432`, except dev/regtest windows documented in `protocol/src/upgrade.rs`. On non-mainnet chains (`chain_id != 0`), `check_gated_*` bypasses these gates — testnets MUST document their own policy.
4. **TEX settlement:** Type3 execution always calls `do_settlement()` after top-level actions (`protocol/src/transaction/type3.rs`). If any TEX transfer cells ran, zhu/sat/dia/asset ledger totals MUST be zero-sum before settlement succeeds.
5. **TEX signatures:** Each `TexCellAct` MUST be signed over `addr + cells` only (replayable across transactions).
6. **Guard topology:** Bare `GUARD` actions MAY appear at top level alongside other top actions. Transactions MUST NOT be guard-only (`protocol/src/action/level.rs`).
7. **Asset TEX gas:** Transactions with asset transfer TEX cells (`cellid` 7 or 8) MUST set `gas_max > 0` and initialize gas (`extra9` path).

Wallets SHOULD:

- Pre-validate TEX zero-sum pairing off-chain before co-signing counterparty bundles.
- Before signing a TEX bundle, verify its cells appear in the **agreed composed Type3 transaction** (structural equality or tx hash), not only that signer `addr` matches.
- Display guard windows (height, chain, balance floor) in human-readable form.
- Reject co-signed TEX bundles whose `addr` does not match the expected counterparty.

Indexers SHOULD:

- Index `TexCellAct` by signer `addr` and settled asset/diamond deltas.
- Classify guard outcomes as **Revert** vs **Fault** and store normalized error text (e.g. `"submitted in height between"`, `"lower than floor"`). There is no stable on-chain guard reason-code enum in v1.
- Correlate issuance (Tx A) and TEX distribution (Tx B) by asset serial, issuer, and block position; index them as a coordinated two-tx event (same block or later), not a single-tx atomic mint+distribute.
- For P5 AST txs, record `ast_branch: if|else` and `cond_outcome: success|revert|fault` on successful transactions (condition revert with else success is not a failed tx).

## 4. Pattern P1 — Atomic multi-asset TEX swap

### 4.1 Intent

Two or more parties atomically exchange HAC, SAT, diamonds, and/or HIP20 assets in one transaction without trusting an escrow contract.

### 4.2 Structure

1. Party A publishes `TexCellAct_A` with pay cells (`cellid` 1/3/5/7) and/or get cells (`2/4/6/8`).
2. Party B publishes `TexCellAct_B` with the mirrored get/pay cells.
3. A coordinator (or either party) combines both signed bundles in one Type3 transaction.
4. Optional funding actions (e.g. `HacToTrs`) MAY precede TEX actions in the same tx. `AssetCreate` is `TOP_ONLY` and MUST use the two-tx P4 flow (§7).

### 4.3 Rules

- MUST: For every pay cell in A, B MUST include the matching get cell (same asset serial, amount, coin type).
- MUST: TEX ledger totals for `zhu`, `sat`, `dia`, and each asset serial MUST be zero before `do_settlement()`.
- MUST: Each party signs only their own bundle; tampering after sign MUST fail verification.
- SHOULD: Include TEX condition cells (`cellid >= 11`) when quoting requires balance/height proofs at execution time.

### 4.4 Failure modes

| Failure | Result |
|---------|--------|
| Imbalanced pay/get | Fault at settlement (`coin` / `asset <n>` / `diamonds settlement check failed`) |
| Bad signature | Fault on `TexCellAct` execute |
| Missing asset / insufficient balance | Fault during cell execute |
| Asset cells without gas | Fault (`gas not initialized`) |

## 5. Pattern P2 — Time-boxed guarded payment

### 5.1 Intent

Release a payment only if the transaction is included within a block-height window.

### 5.2 Structure

```
actions: [
  HeightScope { start, end },   // GUARD, top-level
  HacToTrs | SatToTrs | ...     // non-guard debit
]
```

`end = 0` means unlimited upper bound (per `HeightScope` semantics).

### 5.3 Rules

- MUST: List `HeightScope` **before** the debit action (lower action index runs first).
- MUST: `start <= end` when `end != 0` (otherwise **fault**, not revert).
- MUST: Revert (not fault) when current height is outside `[start, end]` (inclusive).
- SHOULD: Set `end` to a finite deadline for offer expiry.

### 5.4 Wallet pitfall (ordering)

Actions run sequentially. If a debit is listed before `HeightScope` and the guard later
reverts, the **transaction still fails**, but step-by-step simulators may already show the
debit as executed. Wallets MUST NOT present partial action progress as final settlement.
Always validate the full tx atomically (`hip23_p2_transfer_before_guard_still_reverts_outside_window`).

On a full node, `try_execute_tx_by` forks state per transaction and merges only on success
(`chain/src/check.rs`); failed txs do not commit partial mutations. Direct `tx.execute()`
calls in tests or wallet simulators can still observe in-tx partial progress that would not
persist on-chain.

### 5.5 Wallet display

Wallets SHOULD show: “Valid heights: `start` … `end` (inclusive)”.

### 5.6 P2 vs P5

| Aspect | P2 (top-level guard) | P5 (`AstIf` + guard condition) |
|--------|----------------------|----------------------------------|
| Guard revert outside window | **Whole tx fails** | **`br_else` runs**; tx may succeed |
| Use when | Payment must not settle unless in window | Fallback branch or no-op else is desired |
| Wallet display | “Expired — tx failed” | “Condition false — else branch taken” |

## 6. Pattern P3 — BalanceFloor protected transfer

### 6.1 Intent

Prevent a transaction from leaving an address below a minimum HAC/SAT/diamond/asset balance.

### 6.2 Structure

```
actions: [
  <debit action>,               // e.g. HacToTrs
  BalanceFloor { addr, ... }    // GUARD, checks in-tx state at this point
]
```

`BalanceFloor` intentionally inspects **in-transaction state at the guard position**, not pre-tx chain state. Type3 **fee is debited after all actions** — floors do not see post-fee balances unless the integrator models fee in the floor value.

### 6.3 Rules

- MUST: Place `BalanceFloor` **after** debits that should be protected.
- MUST: Specify at least one non-zero floor field (`hacash`, `satoshi`, `diamond`, or `assets`).
- MUST: Revert when any checked dimension is below floor.
- SHOULD: Floor values include expected post-transfer dust retention **and** tx `fee` (and gas-paid HAC if applicable) when minimum **final** on-chain balance is intended.
- SHOULD: Only set non-zero fields for dimensions being protected; zero `hacash` does not guard HAC.

### 6.4 Wallet pitfall (ordering)

If `BalanceFloor` is listed **before** a debit, the guard reads **pre-debit** balances and protection is bypassed while the tx may still succeed (`hip23_p3_floor_before_transfer_checks_pre_debit_state`). Wallets MUST enforce debit-then-floor ordering for P3.

Step-by-step simulators may show debits before a failing floor; the full tx still reverts atomically on-chain (see §5.4).

## 7. Pattern P4 — HIP20 issuance + TEX distribution

### 7.1 Intent

Mint a HIP20 asset in Tx A, then distribute units to counterparties via TEX in Tx B (same block or later).

### 7.2 Structure

`AssetCreate` is `TOP_ONLY` — it MUST be the **sole** top-level action in Tx A (no guards, no TEX, no other actions).  
Distribution therefore uses **two coordinated transactions**:

```
Tx A (issuance):
  actions: [ AssetCreate { metadata, protocol_cost } ]

Tx B (distribution, after Tx A confirms or in a later pass):
  actions: [
    TexCellAct_issuer  (AssetPay cells),
    TexCellAct_counter (AssetGet cells),
  ]
```

### 7.3 Rules

- MUST: Keep `AssetCreate` alone at top level (`TOP_ONLY` topology).
- MUST: Run TEX distribution only after the asset exists on-chain (same block ordering or later block).
- MUST: `issuer` in metadata receives minted supply; issuer SHOULD sign issuer `AssetPay` TEX cells (protocol does not bind `TexCellAct.addr` to `metadata.issuer`).
- MUST: Set `gas_max > 0` on Type3 txs with asset TEX cells.
- MUST: Pay `protocol_cost` **exactly equal** to `genesis::block_reward(height)` at inclusion height; debited from **`tx.main`**, not `metadata.issuer`.
- MUST: On mainnet, asset serial `>= 1025` at `ASSET_ALIVE_HEIGHT` (`765432`); serial ceiling grows with height (see `mint/src/action/asset.rs`).
- SHOULD: Pre-sign TEX bundles for Tx B before broadcasting Tx A.

## 8. Pattern P5 — AST conditional settlement

### 8.1 Intent

Choose between settlement branches based on an on-chain guard predicate without deploying a contract.

### 8.2 Structure

```
actions: [
  AstIf {
    cond:   AstSelect { HeightScope | BalanceFloor | ... },
    br_if:  AstSelect { HacToTrs | TexCellAct | ... },
    br_else: AstSelect { ... }
  }
]
```

### 8.3 Rules

- MUST: Use Type3 with `gas_max > 0` when AST depth > 0.
- MUST: Condition `AstSelect` use `exe_min = exe_max = 1` for single guard predicates.
- MUST: Guard-only `AstSelect` / `AstIf` nodes are invalid at tx topology precheck.
- MUST: Condition guard `Revert` selects `br_else`; condition `Fault` aborts entire `AstIf`.
- SHOULD: Condition `AstSelect` contain guard-only actions (no balance mutations in `cond`).
- SHOULD: Keep branch actions simple (single transfer or TEX bundle) for wallet simulation.

### 8.4 Wallet display

- Show which branch executed: `if` vs `else`.
- Distinguish **condition fault** (invalid range → whole tx fails) from **condition revert** (outside window → else branch).
- Budget gas for **both** condition attempt and branch attempt; AST try costs are not refunded on revert (`doc/ast-spec.md` §6).

### 8.5 Signatures and gas

- `AstIf` collects signatures from `cond`, `br_if`, and `br_else` in the serialized tree — signers for **both** branches MAY be required at broadcast even if only one branch runs.
- Minimum `gas_max` for simple HAC `br_if` + `HeightScope` cond: **17** (template default). TEX or VM branches need higher budgets.

## 9. Security considerations

- **TEX replay:** Signed TEX bundles are replayable; never treat a signature alone as a one-time authorization. Pin the full composed tx before co-signing.
- **Ordering:** Guards only see state mutations from earlier actions in the same transaction.
- **Co-signing:** Verify counterparty TEX cells match your quote before signing.
- **Gas:** AST and asset TEX paths consume gas; under-budgeted txs fail after partial snapshot work.
- **P2 vs P5:** Do not use P5 when a failed guard must abort the entire payment (use P2).

## 10. Versioning

| Version | Contents |
|---------|----------|
| v1.0 | Patterns P1–P5, JSON templates, regression + adversarial tests |
| v1.1 (this draft) | Wallet checklist, indexer dictionary, threat model, invariants, traceability, audit package |
| v2.0 (future) | Optional HVM companion contracts per pattern |

## 11. Test matrix

### Happy path (`hip23_pattern_regression.rs`)

| Pattern | Test |
|---------|------|
| P1 | `hip23_pattern_1_atomic_tex_swap` |
| P2 | `hip23_pattern_2_height_guarded_payment` |
| P3 | `hip23_pattern_3_balance_floor_protected_transfer` |
| P4 | `hip23_pattern_4_asset_create_plus_tex` |
| P5 | `hip23_pattern_5_ast_conditional_settlement` |

### Adversarial (`hip23_pattern_adversarial.rs`)

| Area | Tests |
|------|-------|
| P1 TEX | imbalanced HAC/SAT, tampered sign, insufficient balance, gas required, HAC+SAT swap, height condition cell, HAC prelude + TEX |
| P2 Guard | boundary inclusive, above-end reject, unlimited end, ChainAllow, wrong debit/guard order |
| P3 Floor | asset dimension, satoshi dimension, pre-debit vs post-debit placement |
| P4 HIP20 | duplicate serial, missing asset, issuer insufficient, wrong protocol_cost, `AssetCreate`+TEX same-tx rejected |
| P5 AST | condition fault, else-branch transfer, zero gas rejected |
| Topology | guard-only rejected (precheck + execute), height+floor+transfer combo (pass + fail), height+TEX combo (pass + fail) |

### Stress (`hip23_pattern_stress.rs`)

| Area | Tests |
|------|-------|
| Guards | triple ChainAllow+Height+Floor, height 0..0, far-future start |
| TEX | three-party zero-sum, serial mismatch, empty/unsigned bundles, duplicate addr |
| P3/P4/P5 | multi-debit floor, minsri serial + chained TEX, BalanceFloor AST cond, low gas |
| HIP20 | serial 1025 @ alive height, serial below minsri fault |

### Production path (`hip23_production_path.rs`)

Smoke tests with `fast_sync = false` (`make_ctx_strict` in `[hip23-integration/tests/common/hip23.rs](https://github.com/Moskyera/hip23-integration)`):

| Area | Tests |
|------|-------|
| P1–P5 happy path | `hip23_production_p1_tex_swap_succeeds` … `hip23_production_p5_ast_conditional_settlement` |
| Mempool policy | `hip23_production_duplicate_tx_rejected`, `hip23_production_tampered_main_signature_rejected` |

### Property-based (`hip23_proptest.rs`)

| Property | Cases | Harness |
|----------|-------|---------|
| Balanced HAC TEX settles | 64 | `fast_sync = true` |
| HeightScope `end=0` unbounded above | 64 | `fast_sync = true` |
| HeightScope window (inside ok, outside fail) | 64 | `fast_sync = true` |
| Guard-only always rejected at precheck | 64 | n/a |
| Imbalanced TEX always fails | 64 | `fast_sync = true` |
| Balanced TEX under strict path | 64 | `fast_sync = false` |
| Wrong `protocol_cost` always fails (P4) | 64 | `fast_sync = true` |
| P5 else on height revert | 64 | `fast_sync = true` |
| TEX wire parse never panics | 64 | fuzz-adjacent |

### Guard error codes (`hip23_guard_error_codes.rs`)

Stable string → code mapping for indexers (F-007 mitigation).

### Test vectors (`hip23_test_vectors.rs` + `[hip23-integration/tests/fixtures/hip23_test_vectors.json](https://github.com/Moskyera/hip23-integration)`)

Cross-implementation acceptance registry (12+ vectors).

### Audit strict (`hip23_audit_strict.rs`)

Adversarial cases under `fast_sync = false`: imbalanced TEX, tampered TEX sig, height guard, wrong `protocol_cost`, main sig tamper, guard-only precheck.

### Chain integration (`hip23_chain_integration.rs`)

Per-tx fork/merge semantics (`chain/src/check.rs`): failed tx rollback, P4 Tx A→B sequencing, fail-then-success isolation.

### TEX replay (`hip23_tex_replay.rs`)

TEX signature scope: replay across different `main`, tamper after sign, extra unbalanced party.

Run all:

```bash
cargo test hip23_ -- --nocapture
```

Run production + proptest only:

```bash
cargo test hip23_pro -- --nocapture
```

**Note:** Regression, adversarial, and stress suites use `fast_sync = true` to focus on pattern
semantics (guards, TEX settlement, topology). Production-path and strict proptest suites use
`fast_sync = false` to exercise signature verification and duplicate-tx rejection. Integrators
should run both before shipping wallet or indexer integrations.