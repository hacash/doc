# HIP-23 JSON Templates (Type3)

Templates for wallet builders and integrators. Replace placeholders in `ALL_CAPS`.  
Action JSON uses `kind` (not `type`). TEX cells use `cellid` (not `kind`).

**Normative rules:** `spec.md` (sections linked per pattern below).

**Mainnet:** submit only at `height >= 765432` unless using dev/regtest gates.

---

## Shared placeholders

| Token | Meaning |
|-------|---------|
| `MAIN_ADDR` | Transaction fee payer / primary signer |
| `PARTY_A` / `PARTY_B` | TEX co-signers |
| `RECIPIENT` | Payment destination |
| `SERIAL` | HIP20 asset serial (u64); mainnet ≥ 1025 @ height 765432 |
| `FEE` | Wire amount string, e.g. `"8:244"` |
| `GAS_MAX` | Type3 gas byte: `0` for HAC-only TEX; `17+` for AST; `99` when asset TEX (cells 7/8) |
| `TIMESTAMP` | Unix seconds |
| `START_HEIGHT` / `END_HEIGHT` | Block height guard window (inclusive) |

---

## P1 — Atomic multi-asset TEX swap

See `spec.md` §4.

### Transfer cell reference (cells 1–8)

| cellid | Role | Asset |
|--------|------|-------|
| 1 | ZhuPay | HAC zhu out |
| 2 | ZhuGet | HAC zhu in |
| 3 | SatPay | SAT out |
| 4 | SatGet | SAT in |
| 5 | DiaPay | Named diamonds out |
| 6 | DiaGet | Diamond count in |
| 7 | AssetPay | HIP20 out |
| 8 | AssetGet | HIP20 in |

Counterparty MUST mirror pay↔get with matching amounts/serials.

### Minimal HAC + HIP20 swap (two parties)

```json
{
  "type": 3,
  "main": "MAIN_ADDR",
  "fee": "FEE",
  "gas_max": 99,
  "timestamp": TIMESTAMP,
  "actions": [
    {
      "kind": 22,
      "addr": "PARTY_A",
      "cells": [
        { "cellid": 1, "haczhu": 100000000 },
        { "cellid": 8, "serial": SERIAL, "amount": 50 }
      ],
      "sign": "PARTY_A_TEX_SIGN"
    },
    {
      "kind": 22,
      "addr": "PARTY_B",
      "cells": [
        { "cellid": 2, "haczhu": 100000000 },
        { "cellid": 7, "serial": SERIAL, "amount": 50 }
      ],
      "sign": "PARTY_B_TEX_SIGN"
    }
  ]
}
```

**Notes:** `gas_max: 99` required because cells 7/8 are present. HAC-only swaps MAY use `gas_max: 0`.

### Party A bundle (HAC + SAT + diamonds + asset) — mirror for Party B

Single-party TEX bundle; counterparty MUST use matching get/pay cells (see `tests/tex.rs` `trs1()`).

```json
{
  "kind": 22,
  "addr": "PARTY_A_BUYER",
  "cells": [
    { "cellid": 2, "haczhu": 100000000 },
    { "cellid": 4, "satnum": 2 },
    { "cellid": 6, "dianum": 3 },
    { "cellid": 8, "serial": SERIAL, "amount": 100 }
  ],
  "sign": "PARTY_A_TEX_SIGN"
}
```

```json
{
  "kind": 22,
  "addr": "PARTY_B_SELLER",
  "cells": [
    { "cellid": 1, "haczhu": 100000000 },
    { "cellid": 3, "satnum": 2 },
    { "cellid": 5, "diamonds": "KKKKVA,HYXYHY,UETWNK" },
    { "cellid": 7, "serial": SERIAL, "amount": 100 }
  ],
  "sign": "PARTY_B_TEX_SIGN"
}
```

### Optional HAC funding before TEX

```json
{
  "type": 3,
  "main": "MAIN_ADDR",
  "fee": "FEE",
  "gas_max": 0,
  "timestamp": TIMESTAMP,
  "actions": [
    { "kind": 1, "to": "PARTY_A", "hacash": "10:244" },
    {
      "kind": 22,
      "addr": "PARTY_A",
      "cells": [{ "cellid": 1, "haczhu": 100000000 }],
      "sign": "PARTY_A_TEX_SIGN"
    },
    {
      "kind": 22,
      "addr": "PARTY_B",
      "cells": [{ "cellid": 2, "haczhu": 100000000 }],
      "sign": "PARTY_B_TEX_SIGN"
    }
  ]
}
```

---

## P2 — Time-boxed guarded payment

See `spec.md` §5.

```json
{
  "type": 3,
  "main": "MAIN_ADDR",
  "fee": "FEE",
  "gas_max": 0,
  "timestamp": TIMESTAMP,
  "actions": [
    {
      "kind": 1042,
      "start": START_HEIGHT,
      "end": END_HEIGHT
    },
    {
      "kind": 1,
      "to": "RECIPIENT",
      "hacash": "10:244"
    }
  ]
}
```

**Notes:**

- `kind` `1042` = `HeightScope` (`0x0412`).
- Guard MUST be listed **before** the debit action.
- `end: 0` = no upper height limit.
- `start > end` (when `end != 0`) is a **fault**, not a revert.

### ChainAllow variant

```json
{
  "type": 3,
  "main": "MAIN_ADDR",
  "fee": "FEE",
  "gas_max": 0,
  "timestamp": TIMESTAMP,
  "actions": [
    { "kind": 1041, "chains": [0, 1] },
    { "kind": 1, "to": "RECIPIENT", "hacash": "3:244" }
  ]
}
```

`kind` `1041` = `ChainAllow` (`0x0411`).

---

## P3 — BalanceFloor protected transfer

See `spec.md` §6.

```json
{
  "type": 3,
  "main": "MAIN_ADDR",
  "fee": "FEE",
  "gas_max": 0,
  "timestamp": TIMESTAMP,
  "actions": [
    {
      "kind": 1,
      "to": "RECIPIENT",
      "hacash": "100:244"
    },
    {
      "kind": 1043,
      "addr": "MAIN_ADDR",
      "hacash": "900:244",
      "satoshi": 0,
      "diamond": 0,
      "assets": []
    }
  ]
}
```

**Notes:**

- `kind` `1043` = `BalanceFloor` (`0x0413`).
- Floor is evaluated **after** the preceding debit.
- Include tx `fee` in floor when protecting **final** post-settlement balance.
- Add `assets: [{ "serial": SERIAL, "amount": MIN_AMT }]` to protect HIP20 balances.

### Multi-debit then floor

```json
{
  "actions": [
    { "kind": 1, "to": "RECIPIENT_A", "hacash": "40:244" },
    { "kind": 1, "to": "RECIPIENT_B", "hacash": "10:244" },
    {
      "kind": 1043,
      "addr": "MAIN_ADDR",
      "hacash": "900:244",
      "satoshi": 0,
      "diamond": 0,
      "assets": []
    }
  ]
}
```

---

## P4 — HIP20 issuance + TEX distribution

See `spec.md` §7.

`AssetCreate` is `TOP_ONLY`. Use **two transactions**:

### Tx A — issuance

```json
{
  "type": 3,
  "main": "MAIN_ADDR",
  "fee": "FEE",
  "gas_max": 0,
  "timestamp": TIMESTAMP,
  "actions": [
    {
      "kind": 16,
      "metadata": {
        "serial": SERIAL,
        "supply": 10000,
        "decimal": 2,
        "issuer": "ISSUER_ADDR",
        "ticket": "USDT",
        "name": "Tether"
      },
      "protocol_cost": "BLOCK_REWARD_AT_HEIGHT"
    }
  ]
}
```

**Notes:**

- Tx A MUST contain **only** `AssetCreate` (no guards, no TEX).
- `protocol_cost` MUST equal `genesis::block_reward(height)` at inclusion height (not a fixed literal).
- `MAIN_ADDR` pays `protocol_cost`; `issuer` receives minted supply.

### Tx B — TEX distribution (after Tx A is valid on-chain)

```json
{
  "type": 3,
  "main": "MAIN_ADDR",
  "fee": "FEE",
  "gas_max": 99,
  "timestamp": TIMESTAMP,
  "actions": [
    {
      "kind": 22,
      "addr": "ISSUER_ADDR",
      "cells": [
        { "cellid": 7, "serial": SERIAL, "amount": 500 }
      ],
      "sign": "ISSUER_TEX_SIGN"
    },
    {
      "kind": 22,
      "addr": "RECIPIENT",
      "cells": [
        { "cellid": 8, "serial": SERIAL, "amount": 500 }
      ],
      "sign": "RECIPIENT_TEX_SIGN"
    }
  ]
}
```

---

## P5 — AST conditional settlement

See `spec.md` §8.

Pay only if height guard passes; otherwise fallback else branch:

```json
{
  "type": 3,
  "main": "MAIN_ADDR",
  "fee": "FEE",
  "gas_max": 17,
  "timestamp": TIMESTAMP,
  "actions": [
    {
      "kind": 26,
      "cond": {
        "kind": 25,
        "exe_min": 1,
        "exe_max": 1,
        "actions": [
          { "kind": 1042, "start": START_HEIGHT, "end": END_HEIGHT }
        ]
      },
      "br_if": {
        "kind": 25,
        "exe_min": 1,
        "exe_max": 1,
        "actions": [
          { "kind": 1, "to": "RECIPIENT", "hacash": "5:244" }
        ]
      },
      "br_else": {
        "kind": 25,
        "exe_min": 1,
        "exe_max": 1,
        "actions": [
          { "kind": 1, "to": "RECIPIENT", "hacash": "3:244" }
        ]
      }
    }
  ]
}
```

**Notes:**

- `kind` `26` = `AstIf`, `25` = `AstSelect`.
- Condition guard failure (revert) selects `br_else`; invalid range (`start > end`) **faults** whole node.
- `gas_max` MUST be non-zero for AST execution.
- Signatures are collected from **all** branches in the serialized tree, not only the executed branch.
- Empty else (`exe_min: 0, exe_max: 0, actions: []`) is equivalent to `AstSelect::nop()` in tests.

---

## Composition templates

See `spec.md` §11 topology tests.

### Height + BalanceFloor + transfer (happy path)

```json
{
  "type": 3,
  "main": "MAIN_ADDR",
  "fee": "FEE",
  "gas_max": 0,
  "timestamp": TIMESTAMP,
  "actions": [
    { "kind": 1042, "start": START_HEIGHT, "end": END_HEIGHT },
    { "kind": 1, "to": "RECIPIENT", "hacash": "40:244" },
    {
      "kind": 1043,
      "addr": "MAIN_ADDR",
      "hacash": "900:244",
      "satoshi": 0,
      "diamond": 0,
      "assets": []
    }
  ]
}
```

### Height guard + TEX swap

```json
{
  "type": 3,
  "main": "MAIN_ADDR",
  "fee": "FEE",
  "gas_max": 0,
  "timestamp": TIMESTAMP,
  "actions": [
    { "kind": 1042, "start": START_HEIGHT, "end": END_HEIGHT },
    {
      "kind": 22,
      "addr": "PARTY_A",
      "cells": [{ "cellid": 1, "haczhu": 100000000 }],
      "sign": "PARTY_A_TEX_SIGN"
    },
    {
      "kind": 22,
      "addr": "PARTY_B",
      "cells": [{ "cellid": 2, "haczhu": 100000000 }],
      "sign": "PARTY_B_TEX_SIGN"
    }
  ]
}
```

---

## Guard kind reference

| kind (decimal) | kind (hex) | Action |
|----------------|------------|--------|
| 1041 | 0x0411 | `ChainAllow` |
| 1042 | 0x0412 | `HeightScope` |
| 1043 | 0x0413 | `BalanceFloor` |

---

## TEX condition cell reference (optional)

| cellid | Name | Purpose |
|--------|------|---------|
| 11–13 | Zhu at most / at least / eq | HAC balance conditions |
| 14–16 | Sat conditions | SAT balance conditions |
| 17–19 | Diamond conditions | HACD count conditions |
| 20–22 | Asset conditions | HIP20 balance conditions |
| 23–24 | Height at most / at least | Block height conditions |
| 25 | ChainId eq | Chain ID condition |

Example height condition inside a TEX bundle:

```json
{ "cellid": 23, "height": 800000 }
```

---

## Action kind quick reference

| kind | Action |
|------|--------|
| 1 | `HacToTrs` |
| 16 | `AssetCreate` |
| 22 | `TexCellAct` |
| 25 | `AstSelect` |
| 26 | `AstIf` |
| 1041 | `ChainAllow` |
| 1042 | `HeightScope` |
| 1043 | `BalanceFloor` |