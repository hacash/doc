# HIP-23 Indexer Field Dictionary

Version: 1.0 (HIP-23 v1.1)  
Date: 2026-06-22  
Audience: explorer, indexer, analytics engineers

---

## 1. Transaction-level fields

| Field | Type | When set | Description |
|-------|------|----------|-------------|
| `hip23_pattern` | enum | classifier | `P1`…`P5` or `unknown` |
| `istanbul_gated` | bool | always | `height >= 765432` on mainnet |
| `gas_max` | u8 | Type3 | Gas budget |
| `gas_used` | u8 | Type3 success | Consumed gas |
| `fast_sync_bypass` | bool | dev only | True if node ran without sig checks |

---

## 2. Guard outcome fields

| Field | Type | Values | Notes |
|-------|------|--------|-------|
| `guard_kind` | string | `HeightScope`, `BalanceFloor`, `ChainAllow` | From action type |
| `guard_outcome` | enum | `pass`, `revert`, `fault` | Per guard execution |
| `guard_error_norm` | string | see §3 | Normalized substring match |

**Revert vs fault (MUST classify):**

- **Revert:** expected predicate false (e.g. outside height window) — P2 fails whole tx; P5 may take else.
- **Fault:** invalid parameters or protocol violation (e.g. `start > end`).

---

## 3. Normalized error strings (v1)

Reference implementation: `[hip23-integration/tests/common/hip23_errors.rs](https://github.com/Moskyera/hip23-integration)` (`classify_error`, `error_code_name`).

Indexers SHOULD map raw errors to these buckets:

| Bucket | Substring(s) | `guard_outcome` |
|--------|--------------|-----------------|
| `height_outside_window` | `submitted in height between` | revert |
| `height_invalid_range` | `start height cannot be greater` | fault |
| `balance_below_floor` | `lower than floor` | revert |
| `chain_not_allowed` | `chain id check failed` | revert |
| `guard_only_topology` | `all GUARD` | fault (precheck) |
| `tex_settlement_imbalance` | `settlement check failed` | fault |
| `tex_sig_fail` | `signature verification failed` | fault |
| `gas_not_initialized` | `gas not initialized` | fault |
| `duplicate_tx` | `already exists` | fault |
| `protocol_cost_mismatch` | `protocol cost` | fault |

There is **no** stable on-chain enum in v1 — string matching is required.

---

## 4. TEX fields (P1, P4 Tx B)

| Field | Type | Description |
|-------|------|-------------|
| `tex_signer` | address | `TexCellAct.addr` |
| `tex_cell_ids` | uint[] | Cell types in bundle |
| `tex_zhu_delta` | map addr→i64 | Net zhu change at settlement |
| `tex_sat_delta` | map addr→i64 | Net sat change |
| `tex_dia_delta` | map addr→i32 | Net diamond count change |
| `tex_asset_delta` | map (addr,serial)→i64 | Per-asset net change |
| `tex_settlement_ok` | bool | `do_settlement()` succeeded |

---

## 5. P4 two-tx correlation

| Field | Type | Description |
|-------|------|-------------|
| `hip20_serial` | u64 | Asset serial |
| `hip20_issuer` | address | From `AssetCreate.metadata.issuer` |
| `hip20_tx_role` | enum | `issuance` (Tx A) or `distribution` (Tx B) |
| `hip20_correlation_id` | string | `{serial}:{issuer}` or custom |
| `hip20_same_block` | bool | Tx A and Tx B in same block |

Indexers MUST NOT treat P4 as single atomic tx — store two linked events.

---

## 6. P5 AST fields

| Field | Type | Values |
|-------|------|--------|
| `ast_branch` | enum | `if`, `else`, `none` |
| `cond_outcome` | enum | `success`, `revert`, `fault` |
| `ast_depth` | u8 | Max depth reached |
| `ast_gas_used` | u8 | Gas consumed |

**Important:** `cond_outcome=revert` + `ast_branch=else` + tx success is a **valid** outcome, not a failed tx.

---

## 7. P2 vs P5 classifier hints

| Observation | Likely pattern |
|-------------|----------------|
| Top-level `HeightScope` + transfer, tx failed outside window | P2 |
| `AstIf` + height cond, else transfer, tx succeeded outside window | P5 |
| `ast_branch=else` present | P5 |

---

## 8. Example JSON (indexer output)

```json
{
  "tx_hash": "0x…",
  "hip23_pattern": "P5",
  "ast_branch": "else",
  "cond_outcome": "revert",
  "guard_error_norm": "height_outside_window",
  "tex_settlement_ok": true
}
```