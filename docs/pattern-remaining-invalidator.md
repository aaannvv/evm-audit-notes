# Pattern: Remaining-Amount Invalidator (Value-Packed State Machine)

*Pattern notes from reading 1inch limit-order-protocol's `RemainingInvalidatorLib.sol` + `OrderMixin.sol` fill path. Part of the [working notes](../README.md).*

## The Pattern

Partially-fillable orders need two pieces of state per (maker, orderHash): *is this order alive* and *how much is left*. The obvious design — a struct or two mappings — costs two SSTOREs on first touch. 1inch's design packs both into **one uint256** with three semantics:

```
stored value == 0            → order is NEW (never filled)     [mapping default: zero storage cost]
stored value == ~uint256(0)  → order fully filled / invalidated
stored value == ~remaining   → partially filled, remaining left
```

Reads decode with bitwise NOT: `remaining = ~value` (in `unchecked` — NOT never overflows). Writes store the complement of the new remaining. Because `~0 == maxUint256`, a *partially filled order reaching zero remaining* and an *explicit cancellation* (`fullyFilled()`) converge to the same stored value — one sentinel encodes both "done" paths.

The zero-value trick is the elegant part: Solidity mappings default to zero, so the most common state in a world of millions of orders (fresh, never touched) costs **no storage at all** until first fill.

## What to Check When You See This Pattern

- **Two same-name readers, different failure modes.** `remaining(invalidator)` reverts on `0` ("unknown order"), while `remaining(invalidator, orderMakerAmount)` returns the full amount on `0` ("treat as fresh"). Any external integration that picks the wrong overload gets either a revert loop or, worse, *free full-size order replay* if the writer ever fails to persist a fill. Verify every caller uses the overload whose failure mode matches the trust context.
- **State-as-lock reentrancy guard.** The fill path re-checks `isNewOrder(...)` after callbacks (`OrderMixin.sol:182`): a re-entrant fill of the same order sees a nonzero invalidator and reverts. This works only because the invalidator write happens *before* external calls — confirm ordering on every path, including partial-fill extensions.
- **Complement collisions.** `~remaining` for remaining = 0 equals the fully-filled sentinel — intentional convergence here. But if a variant stores `remaining - 1` or splits the slot into bitfields, the "zero remaining" and "cancelled" cases may diverge, and off-chain indexers will miscount liquidity. Check what each terminal state means for *their* accounting.
- **Unchecked NOT + cast widths.** The library operates on uint256 throughout. Narrow the type (uint128 order amounts packed into a slot) and the complement trick needs re-derivation — high bits of the slot now leak into the decoded remaining.
- **Invalidation races.** Cancellation (`invalidate`) and fills both write the same slot atomically per-tx, but a resolver batching cancel+fill relies on ordering only inside one tx. Cross-tx, an order canceled at block N can still be *simulated* as fillable by off-chain quoters reading stale state — integration-level griefing, not a contract bug, but worth flagging in reports.

## Generalization

"Value-packed state machines" — one storage word encoding both a lifecycle state and a continuous quantity — appear as:

| Variant | Zero means | Complement/sentinel means | Seen in |
| --- | --- | --- | --- |
| Remaining invalidator (this pattern) | fresh order | terminal (filled/cancelled) | 1inch LOP |
| Inverse nonce accumulators | all epochs valid | bitmap of invalidated epochs | Permit2, CowSwap |
| `balanceOf` as allowance proxy | no position | — | ERC-4626 donation variants |

The audit instinct: whenever a single word is doing double duty, enumerate *every* value it can hold, then check that each reader's behavior at each value matches what the writer intended — especially the values that arise from arithmetic convergence (like `~0`) rather than explicit writes.
