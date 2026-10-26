# Pattern 5: Cross-Protocol Address Semantics — When Your Recipient Is Someone Else's Vault

*From a review of a Uniswap v4 hook fee controller (2026-10)*

## The Trap

A fee controller validates its `protocolFeeRecipient` with a blocklist. The obvious
entries are there: `address(0)`, the hook itself, `address(this)`. Then it rejects two
more addresses that look innocent at first glance — the v4 **PoolManager** and the v4
**PositionManager**.

Why? Not because of anything *this* protocol does with them, but because of what
*other* protocols let anyone do with balances parked at those addresses:

- A `take()` to the PoolManager is a self-transfer: the hook's delta is debited, but the
  tokens never leave the manager. The fee is destroyed, not paid.
- The PositionManager's `SWEEP` action is **permissionless**
  (`modifyLiquiditiesWithoutUnlock` needs no NFT, no approval). Its *entire* token
  balance goes to the first caller who sweeps. Fees routed there belong to the first
  bot that shows up.

This came from an external audit finding (V4LE-27): the safety of a payment target is
determined by the target's behavior in **every other protocol that touches it**, not
just your own contract.

## The Rule

> Before an address is accepted as a payment/withdrawal destination, ask: *who else can
> move funds sitting at this address, and under whose permission?*

Three failure classes to check, in order of severity:

1. **First-sweeper wins.** Any contract exposing a permissionless sweep/collect of its
   whole balance (PositionManager-style). Funds are effectively donated to the fastest
   MEV bot.
2. **Frozen forever.** Contracts with no withdrawal path at all for that token
   (Permit2 holding a direct transfer, or a token that arrived without going through
   the contract's accounting). The fee is burned.
3. **Absorbed into someone's settlement.** System contracts that credit the transfer to
   an internal balance of *whoever interacts next* (the hook-itself case: the next
   user's settlement sweeps the hook's balance to them).

## Code Shape

```solidity
function _validateProtocolFeeRecipient(address r) internal view {
    if (
        r == address(0)
            || r == hook
            || r == address(this)
            || r == _hookAddressGetter(IHookSystemContractGetters.poolManager.selector)
            || r == _hookAddressGetter(IHookSystemContractGetters.positionManager.selector)
    ) revert InvalidConfig();
}
```

Two engineering details worth stealing:

- **Read the getters from the target at validation time** (`poolManager()`,
  `positionManager()`), not from hardcoded addresses. The blocklist stays correct across
  deployments.
- **Tolerant staticcall**: the deploy order creates the controller *before* the hook, so
  the constructor cannot rely on the hook having code. Unknown getter or no code → skip
  that check (returns `address(0)`), and re-validate on every later `setProtocolFeeRecipient`.
  Document which guarantees hold only after the hook exists.

## Where This Generalizes

- Fee/treasury recipients in any hook, vault, or marketplace.
- `approve` targets: an allowance to a contract that anyone can trigger is an allowance
  to everyone.
- Flash-loan or rebalance routers: the destination of a leftover-dust `transfer`.
- Airdrop/splitter contracts choosing a fallback beneficiary.

## Audit Checklist Line

For every `transfer`/`take`/`withdraw` destination in the diff: enumerate the external
protocols that treat that address as *their* internal balance or sweepable pool. If any
permissionless exit exists there, the destination is a leak.
