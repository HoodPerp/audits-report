# HoodPerp Smart Contract Security Review

## Auditor

**Pyro** ([@0x3b33](https://x.com/0x3b33))

- Co-founder, [@PhageSec](https://x.com/PhageSec)
- Lead Security Researcher at [@sherlockdefi](https://x.com/sherlockdefi)
- 100+ audits done and over 500 H/M found

## What this is

This report was produced by [Pyro's AI Audit Tool](https://x.com/0x3b33). An audit is a time, resource, and expertise-bound effort in which trained researchers evaluate smart contracts using a combination of manual and automated techniques to identify as many vulnerabilities as possible.

The AI pipeline runs with light human oversight from the auditor above. It is not a substitute for a full professional human audit, but surfaces candidate issues for the team to confirm and remediate. Automated review can miss whole classes of bugs a human would catch, misjudge severity, or flag issues that do not hold up on closer inspection. Treat every finding as a lead to confirm, not a settled fact.

An audit can reveal the presence of vulnerabilities, but it cannot guarantee their absence. This report is a point-in-time assessment of the code at commit `d9a4c97` on branch `main`. It is not a warranty that the system is secure. Use it alongside a professional human audit, a bug bounty, and on-chain monitoring.

## Findings

| ID | Severity | Finding |
|------|----------|---------|
| M-01 | Medium | BuybackVault: minHoodOut is the only on-chain price guard and is unbounded |
| M-02 | Medium | BuybackVault: maxSpendPerRun defaults to 0 (uncapped) |
| L-01 | Low | BuybackVault: interval=0 collapses the TooSoon throttle |
| L-02 | Low | WithdrawRouter: minOut is checked against the Router balance, not the recipient receipt |
| L-03 | Low | Router: swaps the nominal amountIn, not the received balance |
| L-04 | Low | Router: remainder sweep reads raw balanceOf and pays out pre-existing balances |
| L-05 | Low | Router: usdgAssetIndex is immutable and unvalidated |
| L-06 | Low | BuybackVault: checkUpkeep returns empty performData while performUpkeep decodes it |
| L-07 | Low | Router: DustRemaining assumes Lighter.deposit pulls the exact approved amount |
| L-08 | Low | WithdrawRouter: baseline-gated refunds strand donated balances |
| I-01 | Info | checkUpkeep omits the hoodPoolSet gate |
| I-02 | Info | ExactInputSingleParams layout is asserted only by a comment |
| I-03 | Info | No protocol fee is skimmed despite docs describing fees to the buyback |

# [M-01] BuybackVault: minHoodOut is the only price guard and is unbounded on-chain

**Severity: Medium**

## Description
BuybackVault buys HOOD with accrued USDG fee revenue on a keeper cadence. The keeper computes the price off-chain and passes `minHoodOut`. The NatSpec states the contract bounds a compromised keeper "WITHOUT trusting that number" through the owner-set pool, `maxSpendPerRun`, and `interval`.

The output side of the swap is guarded only by `minHoodOut`, and `minHoodOut` is only checked to be nonzero:

```solidity
        if (minHoodOut == 0) revert ZeroMinOut();
```

```solidity
        if (hoodBought < minHoodOut) revert InsufficientOutput(hoodBought, minHoodOut);
```

The same keeper-supplied number is passed as the pool `amountOutMinimum`, and the router's own per-hop price backstop is hardcoded to 0:

```solidity
                amountIn: uint128(amountIn),
                amountOutMinimum: uint128(amountOutMinimum),
                minHopPriceX36: 0,
```

There is no reference price (TWAP or oracle) anywhere in the path. A compromised keeper sets `minHoodOut` to 1 and trades against the USDG/HOOD pool it controls, so the vault spends real USDG for a token amount the keeper chose. The keeper is not-fully-trusted in the protocol's own model, so this defeats a stated safety guarantee. Value at risk is the accrued fee revenue, per run, capped by `maxSpendPerRun` (which is not enforced by default, see the next finding).

## Example
1. Owner sets the USDG/HOOD v4 pool and a keeper, and the vault holds 100k USDG of accrued fees
2. The keeper key is compromised
3. The attacker holds or seeds liquidity in the same USDG/HOOD pool
4. The attacker calls `executeBuyback(1, deadline)` as the keeper
5. `_buyback` swaps up to `maxSpendPerRun` USDG through the pool at a price the attacker sets
6. `hoodBought` is at least 1, so the `InsufficientOutput` check passes
7. The attacker captures the USDG value on the pool side and the vault burns a negligible amount of HOOD

## Recommendation
Do not use a caller-signed floor as the only price guard. Derive a minimum-out on-chain from a Uniswap v4 TWAP of the USDG/HOOD pool with an owner-set maximum deviation, and require the keeper's `minHoodOut` to meet that on-chain floor.

```diff
     function _buyback(uint256 minHoodOut, uint256 deadline) internal {
         if (block.timestamp < lastRun + interval) revert TooSoon(lastRun + interval);
         if (!hoodPoolSet) revert NotConfigured();
         if (minHoodOut == 0) revert ZeroMinOut();
+        // Reference-price floor: reject a keeper minHoodOut below the TWAP-derived minimum.
+        uint256 floor = _twapMinOut(amountIn); // v4 pool TWAP * (1 - maxDeviation)
+        if (minHoodOut < floor) revert MinOutBelowFloor(minHoodOut, floor);
```

# [M-02] BuybackVault: maxSpendPerRun defaults to 0 (uncapped)

**Severity: Medium**

## Description
`maxSpendPerRun` is documented as the cap that bounds a compromised keeper's per-run blast radius. It defaults to 0 and the constructor never sets it. In `_buyback`, 0 means no cap:

```solidity
        uint256 amountIn = (maxSpendPerRun != 0 && balance > maxSpendPerRun) ? maxSpendPerRun : balance;
```

If the owner configures the pool and keeper but does not call `setMaxSpendPerRun`, one run spends the entire USDG balance. This is the opposite of the sibling guards, which fail closed: `hoodPoolSet` starts false and blocks buybacks, and `keeper` starts `address(0)` and blocks keeper calls. The single guard meant to limit per-run damage starts open. It is the blast-radius bound for the previous finding, so a compromised keeper drains the whole accrued balance in one call and the owner loses the multi-interval window to revoke the key.

## Example
1. Owner deploys the vault with the default 1-day interval, calls `setHoodPool` and `setKeeper`, but does not call `setMaxSpendPerRun`
2. Fees accrue to 100k USDG
3. The keeper key is compromised
4. The attacker calls `executeBuyback(1, deadline)` as the keeper
5. `maxSpendPerRun` is 0, so `amountIn` is the full 100k USDG balance
6. The entire balance is swapped in one run at an attacker-chosen price (see the previous finding)

## Recommendation
Make the cap mandatory. Require a nonzero `maxSpendPerRun` at deploy and reject 0 in the setter, so the bound is fail-closed like the others.

```diff
     error ProtectedToken();
+    error ZeroMaxSpend();
```

```diff
     function setMaxSpendPerRun(uint256 _maxSpendPerRun) external onlyOwner {
+        if (_maxSpendPerRun == 0) revert ZeroMaxSpend();
         maxSpendPerRun = _maxSpendPerRun;
         emit MaxSpendPerRunUpdated(_maxSpendPerRun);
     }
```

Set `maxSpendPerRun` in the constructor as well, so a deploy that skips the setter is not left uncapped.

# [L-01] BuybackVault: interval=0 collapses the TooSoon throttle

**Severity: Low**

## Description
`interval` throttles buyback frequency and is one of the three documented bounds on a compromised keeper. The setter and constructor accept 0 without validation:

```solidity
    function setInterval(uint256 _interval) external onlyOwner {
        interval = _interval;
        emit IntervalUpdated(_interval);
    }
```

With `interval == 0`, the throttle `block.timestamp < lastRun + interval` is always false, so the keeper can run the buyback every block. The deploy default is 1 day, so this needs the owner to set 0, but it silently removes a stated bound.

## Recommendation
Reject a zero interval in the setter and the constructor.

```diff
     function setInterval(uint256 _interval) external onlyOwner {
+        if (_interval == 0) revert ZeroInterval();
         interval = _interval;
         emit IntervalUpdated(_interval);
     }
```

# [L-02] WithdrawRouter: minOut is checked against the Router balance, not the recipient receipt

**Severity: Low**

## Description
`convertAndSend` measures the swap output as the Router's own balance delta, enforces `minOut` against it, then transfers to the user:

```solidity
        amountOut = target.balanceOf(address(this)) - targetBefore;

        if (amountOut < minOut) revert InsufficientOutput(amountOut, minOut);
```

```solidity
        target.safeTransfer(msg.sender, amountOut);
```

If the owner whitelists a fee-on-transfer target asset, the `safeTransfer` delivers less than `amountOut`, so the user receives below `minOut` while the check passes. The documented target set is equities and ETFs, which are not fee-on-transfer, so this only occurs under an out-of-set whitelist.

## Recommendation
Measure the recipient delta, or reject fee-on-transfer assets at whitelist time. If fee-on-transfer support is not intended, restrict `setTargetAsset` to standard ERC20s and document it.

# [L-03] Router: swaps the nominal amountIn, not the received balance

**Severity: Low**

## Description
`convertAndDeposit` pulls `amountIn`, then approves and swaps the same nominal `amountIn`:

```solidity
        token.safeTransferFrom(msg.sender, address(this), amountIn);

        // Authorize the Universal Router to pull the stock token from THIS contract via Permit2.
        token.forceApprove(address(permit2), amountIn);
        permit2.approve(stockToken, address(universalRouter), uint160(amountIn), MAX_EXPIRATION);
```

If the owner whitelists a fee-on-transfer stock token, the Router receives less than `amountIn` but tries to settle `amountIn` in the swap, so `SETTLE_ALL` cannot pull the full amount and the call reverts. This is a per-token liveness DoS with no fund loss (the transaction is atomic). Stock tokens are not fee-on-transfer, so it needs an out-of-set whitelist.

## Recommendation
Swap the received balance delta instead of the nominal amount, or reject fee-on-transfer tokens in `setStockToken`.

# [L-04] Router: remainder sweep reads raw balanceOf and pays out pre-existing balances

**Severity: Low**

## Description
After depositing, the Router returns leftover stock token by reading the raw balance, with no pre-call baseline:

```solidity
        // Return any unswapped stock-token remainder to the caller (robustness vs partial fills).
        uint256 tokenLeft = token.balanceOf(address(this));
        if (tokenLeft > 0) token.safeTransfer(msg.sender, tokenLeft);
```

The USDG leg snapshots `usdgBefore`, but this leg does not. Stock token sitting in the Router before the call, from an accidental transfer by a third party, is swept to whoever next calls `convertAndDeposit` for that token. Normal flow leaves zero remainder because the exact-in swap consumes `amountIn` via `SETTLE_ALL`, so only donated balances are at risk and no attacker can force them to exist.

## Recommendation
Snapshot the stock-token balance before the pull and refund only the delta.

```diff
+        uint256 tokenBefore = token.balanceOf(address(this));
         token.safeTransferFrom(msg.sender, address(this), amountIn);
```

```diff
-        uint256 tokenLeft = token.balanceOf(address(this));
-        if (tokenLeft > 0) token.safeTransfer(msg.sender, tokenLeft);
+        uint256 tokenLeft = token.balanceOf(address(this));
+        if (tokenLeft > tokenBefore) token.safeTransfer(msg.sender, tokenLeft - tokenBefore);
```

# [L-05] Router: usdgAssetIndex is immutable and unvalidated

**Severity: Low**

## Description
`usdgAssetIndex` is set once at deploy, is immutable, and is passed to `lighter.deposit` on every call:

```solidity
        usdgAssetIndex = _usdgAssetIndex;
```

```solidity
        lighter.deposit(msg.sender, usdgAssetIndex, ROUTE_PERPS, usdgDeposited);
```

If Lighter reassigns or deprecates the USDG asset id, every `convertAndDeposit` reverts and the Router cannot be updated without a redeploy. This is an external-dependency liveness risk, not a fund loss, and it is not attacker-triggerable.

## Recommendation
Make `usdgAssetIndex` an owner-settable value, or document the redeploy dependency on Lighter keeping the USDG asset id stable.

# [L-06] BuybackVault: checkUpkeep returns empty performData while performUpkeep decodes it

**Severity: Low**

## Description
`performUpkeep` decodes two words from `performData`:

```solidity
        (uint256 minHoodOut, uint256 deadline) = abi.decode(performData, (uint256, uint256));
```

`checkUpkeep` always returns empty `performData`:

```solidity
        performData = "";
```

A standard Chainlink Automation forwarder relays the `checkUpkeep` output into `performUpkeep`, which would `abi.decode` empty bytes and revert. In practice the keeper uses the direct `executeBuyback` path with explicit arguments, and `minHoodOut` must be computed off-chain, which a `view` `checkUpkeep` cannot do, so the forwarder path is not usable here. No fund impact, but the two functions disagree on the `performData` contract.

## Recommendation
Drop the unused Automation interface, or make the two functions agree: have `checkUpkeep` return the encoded params and guard `performUpkeep` against short `performData`.

# [L-07] Router: DustRemaining assumes Lighter.deposit pulls the exact approved amount

**Severity: Low**

## Description
The no-custody check requires the USDG balance not to exceed the pre-call baseline after the deposit:

```solidity
        usdg.forceApprove(address(lighter), usdgDeposited);
        lighter.deposit(msg.sender, usdgAssetIndex, ROUTE_PERPS, usdgDeposited);

        // No-custody invariant: no USDG should remain beyond what was here before.
        if (usdg.balanceOf(address(this)) > usdgBefore) {
            revert DustRemaining(usdg.balanceOf(address(this)) - usdgBefore);
        }
```

This assumes `lighter.deposit` pulls exactly `usdgDeposited`. If Lighter ever pulls less, the residual trips `DustRemaining` and the whole call reverts. This is liveness only (the atomic revert returns the user's stock token) and requires the external Lighter contract to deviate from its documented exact-amount deposit.

## Recommendation
If a partial pull by Lighter is possible, refund the residual to the caller instead of reverting. Otherwise document the exact-pull dependency.

# [L-08] WithdrawRouter: baseline-gated refunds strand donated balances

**Severity: Low**

## Description
`convertAndSend` refunds only the USDG above the pre-call baseline:

```solidity
        uint256 usdgNow = usdg.balanceOf(address(this));
        if (usdgNow > usdgBefore) usdg.safeTransfer(msg.sender, usdgNow - usdgBefore);
```

USDG or target asset sent to the WithdrawRouter out of band, before a call, sits at or below the baseline and is never refunded. There is no rescue path, so accidental donations are stranded. Only self-inflicted donations are affected and no user or protocol funds are at risk. The baseline gating is intentional and protective, because it prevents handing a prior balance to the next caller (the inverse of the Router remainder-sweep finding).

## Recommendation
Accept this as intended no-custody behavior, or add an owner rescue for non-core tokens accidentally sent to the contract.

# Informational

## [I-01] checkUpkeep omits the hoodPoolSet gate
`checkUpkeep` computes `upkeepNeeded` from time and balance only and omits the `hoodPoolSet` gate that `_buyback` enforces at line 132:

```solidity
        upkeepNeeded = (block.timestamp >= lastRun + interval) && usdg.balanceOf(address(this)) > 0;
```

Before `setHoodPool` is called, `checkUpkeep` can report `true` while `performUpkeep` reverts `NotConfigured`. It self-heals once the pool is set and has no fund impact.

```diff
-        upkeepNeeded = (block.timestamp >= lastRun + interval) && usdg.balanceOf(address(this)) > 0;
+        upkeepNeeded = hoodPoolSet && (block.timestamp >= lastRun + interval) && usdg.balanceOf(address(this)) > 0;
```

## [I-02] ExactInputSingleParams layout is asserted only by a comment
`UniversalRouterLib.ExactInputSingleParams` adds a `minHopPriceX36` field before `hookData` relative to canonical Uniswap v4-periphery, justified by a comment rather than a test on the real router. If the layout is wrong for the deployed Robinhood v4 router, swaps misencode. There is no fund-loss path, because all three callers re-check the output with a balance-delta floor, so a bad encoding reverts. Keep a fork test that exercises the live Robinhood v4 router as a CI gate on this struct layout rather than relying on the comment.

## [I-03] No protocol fee is skimmed despite docs describing fees to the buyback
Neither `Router.convertAndDeposit` nor `WithdrawRouter.convertAndSend` skims a protocol fee. The full swap output goes to the user or their Lighter account, while the docs describe protocol fees flowing to the BuybackVault. This is a docs-versus-code gap with no security impact, and the missing fee favors users. Align the docs with the current no-fee implementation, or confirm the fee is taken off-chain or on the Lighter side.
