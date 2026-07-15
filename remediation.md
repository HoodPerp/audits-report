# BuybackVault Remediation Report

**Status:** Implemented in `contract/src/BuybackVault.sol`  
**Audit baseline:** commit `d9a4c97` · Original report: [report.md](./report.md)  
**Scope of this document:** Medium and Low findings on **BuybackVault** only

---

## Summary

The audit identified that a **compromised keeper** could drain the vault's USDG at a bad price because:

- `minHoodOut` was entirely keeper-supplied with no on-chain floor
- `maxSpendPerRun` defaulted to `0` (unlimited per run)
- `interval` could be set to `0` (no throttle)
- Chainlink Automation entrypoints were inconsistent and unusable

We hardened the vault so keeper compromise has **bounded blast radius** and **minimum swap quality** enforced on-chain.

---

## Findings addressed

| ID | Severity | Issue | Resolution |
|---|---|---|---|
| **M-01** | Medium | Keeper-only `minHoodOut` — no on-chain price guard | Owner-set `minHoodPerUsdg` floor; keeper value must meet `(amountIn × minHoodPerUsdg) / 1e6` |
| **M-02** | Medium | `maxSpendPerRun` default `0` = uncapped | Required in constructor; `setMaxSpendPerRun(0)` reverts |
| **L-01** | Low | `interval = 0` disables throttle | Constructor and setter reject zero interval |
| **L-06** | Low | `checkUpkeep` / `performUpkeep` disagree on `performData` | Removed Automation interface; keeper uses `executeBuyback` only |
| **I-01** | Info | `checkUpkeep` omitted `hoodPoolSet` | Removed with Automation hooks |

---

## Solution design

### 1. On-chain price floor (`minHoodPerUsdg`)

**Problem:** A compromised keeper could call `executeBuyback(1, deadline)` and accept almost no HOOD while spending real USDG.

**Logic:**

```
onChainFloor = (amountIn * minHoodPerUsdg) / 1e6
require minHoodOut >= onChainFloor
```

- `minHoodPerUsdg` = minimum HOOD wei (18 decimals) per 1 USDG micro-unit (6 decimals)
- Set by owner via `setMinHoodPerUsdg` before buybacks can run (`minRateSet` gate)
- Keeper still supplies live `minHoodOut` from off-chain quotes; contract rejects quotes below the floor

**Example:** `minHoodPerUsdg = 4e17` (0.4 HOOD per $1). Spending 10,000 USDG requires `minHoodOut >= 4,000 HOOD`.

**Trade-off:** Floor is owner-set, not a live TWAP oracle. Multisig should set a conservative rate and update via timelock when markets move. This matches the audit's intent without adding v4 oracle dependencies.

### 2. Mandatory spend cap (`maxSpendPerRun`)

**Problem:** Default `0` meant "spend entire balance" — one bad keeper tx could drain all accrued fees.

**Logic:**

- Constructor requires `_maxSpendPerRun > 0`
- Each run: `amountIn = min(balance, maxSpendPerRun)`
- Owner can lower cap via `setMaxSpendPerRun`; zero always reverts

Deploy env: `BUYBACK_MAX_SPEND` (USDG base units, 6 decimals).

### 3. Interval throttle

**Problem:** Owner could accidentally set `interval = 0`, allowing buybacks every block.

**Logic:** Constructor and `setInterval` revert on zero.

### 4. Single keeper entrypoint

**Problem:** `performUpkeep` expected encoded `(minHoodOut, deadline)` but `checkUpkeep` returned empty bytes — Automation path always reverted. `minHoodOut` cannot be computed in a `view` function anyway.

**Logic:** Removed `checkUpkeep` and `performUpkeep`. Only:

```solidity
executeBuyback(uint256 minHoodOut, uint256 deadline)
```

Keeper bot polls vault balance + interval off-chain and submits txs directly.

### 5. Configuration gates (unchanged, now stricter)

Buyback reverts `NotConfigured` unless **both**:

- `hoodPoolSet` — owner configured USDG/HOOD v4 pool params
- `minRateSet` — owner configured `minHoodPerUsdg`

---

## Buyback flow (after remediation)

```
Fees accrue as USDG in BuybackVault
        ↓
Keeper waits until block.timestamp >= lastRun + interval
        ↓
Keeper reads live HOOD price off-chain
        ↓
Keeper calls executeBuyback(minHoodOut, deadline)
        ↓
Contract checks:
  • onlyKeeper
  • hoodPoolSet && minRateSet
  • minHoodOut >= onChainFloor
  • amountIn <= maxSpendPerRun
        ↓
Swap USDG → HOOD via owner-set v4 pool (Universal Router)
        ↓
Verify hoodBought >= minHoodOut
        ↓
Transfer HOOD to 0x…dEaD (burn)
        ↓
Emit BuybackExecuted
```

---

## Deploy checklist

1. Launch $HPERP token; note contract address
2. Create USDG/HPERP Uniswap v4 pool on Robinhood Chain
3. Set `HOOD_TOKEN`, `BUYBACK_MAX_SPEND` in `contract/.env`
4. Deploy via `DeployVault.s.sol` (passes token + cap + interval in constructor)
5. Owner txs:
   - `setHoodPool(fee, tickSpacing, hooks)`
   - `setMinHoodPerUsdg(rate)` — conservative floor
   - `setKeeper(keeperAddress)`
6. Fund vault with USDG fee revenue; keeper runs on cadence

---

## Tests

15 unit tests in `contract/test/BuybackVault.t.sol`, including:

- Happy-path buyback and burn
- Spend cap enforcement
- `MinOutBelowFloor` revert
- Zero cap / zero interval reverts
- Not configured (pool / rate) reverts

Run: `forge test --match-contract BuybackVaultTest`

---

## Findings not addressed in this pass

Router and WithdrawRouter items (L-02–L-05, L-07–L-08, I-02–I-03) remain open. They are lower severity, mostly edge cases (fee-on-transfer tokens, accidental donations, external Lighter dependency). Tracked for a follow-up remediation sprint.

---

## Files changed

| File | Change |
|---|---|
| `contract/src/BuybackVault.sol` | Floor rate, mandatory cap, interval validation, removed Automation |
| `contract/test/BuybackVault.t.sol` | Updated + new revert tests |
| `contract/script/DeployVault.s.sol` | Constructor takes `maxSpendPerRun` |
| `contract/.env.example` | Added `BUYBACK_MAX_SPEND` |
