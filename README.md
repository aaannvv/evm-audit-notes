# EVM Audit Working Notes

Practical, field-tested methodology for auditing Solidity protocols — built from full read-throughs of production codebases across five protocols and four pool types (staking, lending markets, multi-asset curve vaults, limit-order matching), roughly 10,000 lines of Solidity reviewed end-to-end.

This is a working document, not a checklist dump. Each section reflects what actually surfaced risk in practice.

## 1. Where Bugs Actually Live: The Thin-Zone Model

Auditing effort is not uniform across a codebase. Vulnerability density concentrates in **thin zones**, which are the intersection of three factors:

- **Edge surfaces** — cross-contract boundaries, bridges, oracles, adapters, routers
- **Recent additions** — new deployments, migrated logic, newly added hooks (note: *new deployment ≠ new code*; re-deployed legacy code inherits its old audit surface)
- **Audit under-exposure** — diff-only audits, time-boxed reviews, components that shipped after the main audit

A function that is two of three (edge + recent) deserves disproportionate attention. A pure, well-copied OpenZeppelin derivative in a five-year-old codebase deserves almost none.

**Signal worth tracking:** "asset refresh days." When a protocol adds new assets/markets, the *listing logic* (oracle wiring, risk parameters, fee routing) is frequently newer than the vault code that consumes it.

## 2. The Seven Attack-Vector Classes

Every state-changing function gets tested against this grid (in order of historical yield):

1. **Access control** — who *can actually* reach this path? (Roles, initializers, upgrade paths; the answer is often not what the natspec claims)
2. **Reentrancy** — external calls before state finalization, especially across the CEI boundary in adapters and routers; also cross-function and read-only reentrancy in view functions consumed by oracles/lending
3. **Numeric integrity** — share/asset math (rounding direction favors protocol or user?), first-depositor inflation, zero-share edge cases, fee accumulation ordering, `unchecked` blocks and their actual overflow assumptions
4. **Input validation** — unvalidated arrays (length mismatch, duplicates), calldata length edge cases, path/token assumptions, address(0)
5. **State consistency** — pause/epoch/migration windows, partial-failure states (what if call 2 of 3 reverts?), accounting drift between bookkeeping and reality
6. **Trust boundaries** — what does this contract assume about its neighbors? Messenger contracts, bridge adapters, oracle feeds, EOA-callback assumptions
7. **Economic/incentive** — who profits from griefing, and is the gas cost of griefing lower than the damage caused?

## 3. Shared-Pool Checklist

For any pooled-asset system (vaults, staking, markets):

- [ ] First depositor / donation inflation attack surface
- [ ] Rounding: does the direction of every division favor the pool (not the depositor)?
- [ ] Withdrawal ordering under insolvency — FIFO, pro-rata, or tombstone?
- [ ] Fee-on-transfer / deflationary token behavior (if third-party tokens accepted)
- [ ] Accounting entry point: is bookkeeping called on *every* asset mutation path, including donations and rescue functions?
- [ ] Pause semantics: what exactly is frozen — deposits, withdrawals, or both asymmetrically?
- [ ] Migration/upgrade path: can shares be carried across? Who executes, and what breaks mid-migration?

## 4. Hook-Chain Matrix

Modern protocols (AA wallets, ERC-4626 wrappers, escrows) compose user hooks. For each hook-bearing contract:

- Can the hook **reenter the host** (same function or another)?
- Can the hook **observe state mid-transaction** and relay it (flashloan-style oracle pollution)?
- Who can **register/replace the hook**, and is that permission revocable?
- Does the hook's **gas stipend** assumption hold under worst-case callee behavior?
- Is the hook **rewarded** (incentivizing attacker registration)?

## 5. Deployment Reality Checks

- `initialize()` protection on implementation contracts behind proxies
- Storage layout collisions across upgrade versions (gap exhaustion)
- Constructor-injected vs. setter-injected dependencies (the latter expand the admin attack surface)
- Immutables that encode assumptions the rest of the codebase may later violate

## 6. Report Integrity

A finding without a precise failure mode is noise. Every candidate finding gets:

1. The exact transaction sequence that triggers it
2. The exact loss (who loses what, quantified)
3. Why existing protections (checks, roles, pauses) fail to prevent it
4. A severity honest enough to survive triage — overstated severities burn reviewer credibility faster than missed low-sev issues

## Pattern Notes (docs/)

Concrete pattern write-ups extracted from the codebases below — what the pattern is, and what to check when you meet it:

- [Bad-Debt Socialization via Earnings Accumulator](docs/pattern-bad-debt-socialization.md) — Exactly's `clearBadDebt`: partial-clear skip semantics, rounding on yield routing, Auditor trust concentration, and the three bad-debt endgames compared.
- [Remaining-Amount Invalidator](docs/pattern-remaining-invalidator.md) — 1inch's `RemainingInvalidatorLib`: one uint256 encoding lifecycle + quantity, arithmetic-convergence sentinels, and the overload failure-mode trap.

## Codebases Worked

Full read-throughs (primary contracts, comment-level): The Graph L1 staking/disputes/curation suite (~5,500 LOC), Exactly protocol markets/auditor (~1,700), 1inch limit-order contracts (~700), 0xIntuition MultiVault/AtomWallet/TrustBonding suite (~3,600), plus supporting adapters and routers.

---

*Working notes; updated as coverage expands. Corrections welcome via issue.*
