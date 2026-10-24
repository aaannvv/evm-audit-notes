# Pattern: Bridge-Fee Pre-Quote on a Representative Amount

*Pattern notes from reading 0xIntuition's `TrustSwapAndBridgeRouter.sol` (swap-and-bridge flows). Part of the [working notes](../README.md).*

## The Pattern

A swap-then-bridge router must pay the bridge's `quoteTransferRemote` fee in native gas token, but the true output amount of the swap is only known *after* the swap executes. The circularity — fee needed before swap, amount known after — is commonly broken by quoting the fee on a **representative amount** (the user's `minTrustOut` floor) rather than the eventual output:

```
bridgeFee = hub.quoteTransferRemote(domain, recipient, minTrustOut)   // pre-swap, non-final amount
...
amountOut = swap(...)                                                  // real output
_bridge(amountOut, recipient, bridgeFee)                               // fee from the pre-quote
```

The router then *assumes* the quoted fee is amount-independent for any non-zero amount (in the Caldera Metallayer case, the hub documents this). The in-code comment says it plainly: the construction "is sound only because" the fee is amount-independent.

## What to Check When You See This Pattern

- **Where does the assumption live?** Here it lives in an external system's *documentation*, not in code or on-chain config. Every assumption imported from docs deserves one question: what breaks if the hub changes fee policy to amount-dependent tomorrow? If `quote(amountOut) > quote(minTrustOut)`, the router underpays the bridge — does the bridge call revert (user loses gas, a griefing surface), or does it silently consume from the transferred amount (user loses tokens)?
- **Quote-then-act race.** Between `quoteTransferRemote` and the actual bridge call, the swap executes. If fee quotes can move intra-block (they usually can't on deterministic fee schedulers, but can on market-based relayers), the quoted fee is stale. Check whether the bridge API distinguishes "quoted" from "required at execution."
- **Dust accounting on the difference.** The ERC20 variant refunds `msg.value - bridgeFee` to the caller. If the refund path (`receive()` + `_refundExcess`) is on a contract without a payable fallback, the whole flow bricks for that integrator — the comment itself warns about this. Fee plumbing and refund plumbing must be audited as one unit.
- **`minTrustOut == 0` guards.** The pre-quote needs a non-zero representative amount, which is why these routers enforce `minTrustOut != 0`. That couples a slippage parameter to a fee-mechanics parameter — users who set loose slippage (minTrustOut very low) get fees quoted on an unrepresentatively small amount. Same assumption, one more way to violate it.
- **Who eats the variance?** If the real `amountOut` exceeds `minTrustOut` and fees were amount-*dependent*, the protocol either reverts or the recipient receives less than the full swap output. Enumerate which party bears each failure: caller (gas), recipient (principal), router (stuck funds).

## Generalization

"Quote on representative input, execute on real input" appears wherever a pre-action requires payment sized by a post-action quantity:

| Instance | Representative input used | Real value diverges when |
| --- | --- | --- |
| Bridge fee pre-quote (this pattern) | user's min-out floor | fee becomes amount-dependent |
| Slippage-checked swap via off-chain quote | off-chain price | oracle/quote staleness intra-block |
| Gas-estimation prepay (gasprice oracles) | current estimate | gas spikes mid-tx |
| Cross-chain message fee on payload size | estimated payload | payload built after quote |

The audit instinct: whenever a fee or limit is computed from a value the contract does *not yet have*, find the exact source of the amount-independence assumption — code, config, or docs — and stress-test what happens on the day that assumption expires.
