# Pattern: Bad-Debt Socialization via Earnings Accumulator

*Pattern notes from reading Exactly Protocol's `Market.sol` (`clearBadDebt`, ~line 605). Part of the [working notes](../README.md).*

## The Pattern

When a borrower's collateral is exhausted by liquidation, the remaining debt doesn't vanish and isn't written off against a treasury. It is **socialized into the floating pool's earnings accumulator**: future floating-pool earnings absorb the loss, diluting all floating depositors' yield rather than any single party's principal.

Core flow (Exactly's implementation):

1. **Gated entry** — only the `Auditor` can call `clearBadDebt` (the Auditor itself is invoked at the end of every liquidation via `handleBadDebt`, which decides when an account is underwater enough to clear).
2. **Fixed-rate legs first** — the borrower's fixed borrow positions are iterated via a **packed 32-bit maturity bitmap**. For each maturity, if `earningsAccumulator >= badDebt`, the position is fully cleared.
3. **Backup-debt yield routing** — when a fixed pool's `repay` returns a backup-debt reduction, the corresponding share of that pool's `unassignedEarnings` is moved into the earnings accumulator (proportional, `mulDivDown`).
4. **Floating leg last** — remaining floating debt is cleared through `previewRepay` + `noTransferRefund`.
5. **Settlement** — `earningsAccumulator -= totalBadDebt`, one `SpreadBadDebt` event.

## What to Check When You See This Pattern

- **Partial-clear skip semantics.** If the accumulator can't cover a maturity's bad debt, that position is *skipped, not reverted*. Clearing may require multiple calls as earnings accrue. Ask: can bad debt sit uncleared indefinitely if the accumulator stays thin? Does any accounting assume it's gone (e.g., market utilization, oracle-fed risk metrics)?
- **Rounding direction on yield routing.** The `mulDivDown` on unassigned-earnings transfer rounds *against* the accumulator being credited (conservative). Invert the direction anywhere in the loop and each clearBadDebt call leaks dust the other way.
- **Bitmap iteration bounds.** The maturity walk advances `maturity += INTERVAL` per shift and stops when the packed word is zero. Positions packed beyond the intended window, or a corrupted bitmap, change loop cost and touched storage — the input-validation question is "who can ever write this bitmap, and do they validate packing?"
- **Trust concentration.** The Auditor is the sole entry point for debt clearance across every Market. An Auditor-level flaw (e.g., clearing debt for a borrower who isn't actually underwater) mints solvency out of the accumulator in every market at once. When auditing one Market, always pull the Auditor's validation into scope.
- **Reward hooks inside the loop.** `handleRewards(true, borrower)` fires per cleared maturity — rewards accounting on an account mid-liquidation is a classic cross-function reentrancy surface if the hook target is attacker-influenced.
- **Event fidelity.** Each cleared maturity emits `RepayAtMaturity` with `assets = badDebt` — downstream subgraphs that treat this as a *real repay* (liquidity returning) will mis-model the pool. Check off-chain consumers before dismissing it as cosmetic.

## Generalization

Socialize-the-earnings is one of three common bad-debt endgames:

| Endgame | Loss lands on | Fragile when |
| --- | --- | --- |
| Earnings accumulator (Exactly) | floating depositors' future yield | earnings < bad debt for long periods |
| Mint-backstop / insurance pool | backstop LPs | backstop drains; incentive to drain |
| Inflate share price (protocol debt token) | all shareholders at once | death-spiral correlation with the loss event |

The audit question is always *who is the residual claimant*, and whether that party has consented (via disclosed docs) to holding that tail risk.
