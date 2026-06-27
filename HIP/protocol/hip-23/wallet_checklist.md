# HIP-23 Wallet Integration Checklist

Version: 1.0 (HIP-23 v1.1)  
Date: 2026-06-22  
Use: pre-sign validation before broadcast or co-sign

---

## 0. Universal (all patterns)

- [ ] Transaction `type >= 3`
- [ ] Mainnet `height >= 765432` (or documented testnet policy)
- [ ] `gas_max > 0` if AST depth > 0 **or** asset TEX cells (7/8) present
- [ ] Not guard-only topology (at least one non-guard top action)
- [ ] Simulate **full tx atomically** — do not show per-action finality on failure
- [ ] Collect all required signatures (`req_sign` / branch signers for P5)
- [ ] Fee payer (`tx.main`) has balance for `fee` + debits + gas

---

## P1 — Atomic TEX swap

- [ ] Every pay cell in party A has matching get in party B (type, serial, amount)
- [ ] Off-chain zero-sum check per dimension (zhu, sat, dia, each asset serial)
- [ ] Each `TexCellAct.addr` matches expected counterparty address
- [ ] Counterparty bundle verified **inside agreed composed Type3** (hash or structural equality)
- [ ] No post-quote tampering of cells (re-verify signatures)
- [ ] Fund pay-side balances before broadcast
- [ ] If asset cells present: `gas_max > 0`

---

## P2 — Height-guarded payment

- [ ] `HeightScope` listed **before** debit action
- [ ] `start <= end` when `end != 0`
- [ ] Display: “Valid heights: `start`…`end` (inclusive)”
- [ ] Confirm P2 (not P5) if expiry must abort entire payment
- [ ] `gas_max = 0` allowed for plain HAC transfer without asset TEX

---

## P3 — BalanceFloor protected transfer

- [ ] Debit action(s) listed **before** `BalanceFloor`
- [ ] At least one non-zero floor field for dimensions being protected
- [ ] Floor value accounts for intended post-tx balance **and** `fee` (and gas if applicable)
- [ ] Zero `hacash` floor does not protect HAC — set explicit fields only

---

## P4 — HIP20 issuance + TEX distribution

- [ ] **Tx A:** `AssetCreate` is the **only** top-level action
- [ ] `protocol_cost == block_reward(expected_height)` exactly
- [ ] `tx.main` holds `protocol_cost` + `fee`
- [ ] Asset serial ≥ minsri at target height (mainnet)
- [ ] **Tx B:** only after Tx A confirmed (or same-block later ordering)
- [ ] Issuer signs issuer `AssetPay` TEX cells
- [ ] `gas_max > 0` on Tx B
- [ ] Pre-sign Tx B bundles before broadcasting Tx A (recommended)

---

## P5 — AST conditional settlement

- [ ] `gas_max > 0` (minimum 17 for simple HeightScope + HAC branch)
- [ ] Cond `AstSelect`: `exe_min = exe_max = 1`
- [ ] Cond contains guard-only actions (no debits in cond)
- [ ] UX: distinguish condition **fault** (tx fails) vs **revert** (else runs)
- [ ] Signatures collected for both branches if required by `req_sign`
- [ ] Budget gas for cond + selected branch attempt

---

## Co-signing workflow (P1 / P4 Tx B)

1. Agree on full action list, order, `fee`, `gas_max`, `main`.
2. Hash composed unsigned tx (or canonical serialization).
3. Each party signs TEX over agreed cells only after step 1–2 locked.
4. Coordinator assembles; each party re-validates final wire tx before broadcast.

---

## Pre-broadcast simulation modes

| Mode | Checks |
|------|--------|
| Fast preview | Pattern semantics only (`fast_sync` equivalent) |
| Production | Signatures, duplicate-tx, fee rules (`fast_sync=false`) |

Run repo tests before shipping wallet integration:

```bash
cargo test hip23_ -- --nocapture
```