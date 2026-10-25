# Method: Auditing a Heavily-Reviewed Contest Codebase

*Method notes from a full pass over a live $50K audit competition codebase (~3,600 lines across vault, oracle, fee, and automation modules) that had already absorbed multiple audit rounds and a four-digit findings count. Part of the [working notes](../README.md).*

## The Situation

Contest codebases that shipped after several audit rounds are the norm at the top platforms. The naive expectation — "everything is found" — is mostly right: across ~3,600 lines and two serious candidate findings, both were rejected by their own math (see below), for zero confirmed findings. That outcome is *information*: it calibrates how much more time the codebase deserves. The method below is what makes that call efficiently.

## 1. Read the Repair-Trace Markers First

A reviewed codebase annotates itself. Four marker families, each with a different meaning:

| Marker | Example | Meaning for you |
| --- | --- | --- |
| Fixed-finding IDs | `V4LE-61`, `(...V4LE-113)` | Known-weak spot, fixed. Re-check the fix's completeness, not the original bug |
| Auditor IDs | `M-3`, `L-h` | Same, from a named external review |
| Accepted-risk declarations | `AUDIT-ACCEPTED-ORACLE-LIQUIDATION-LIVENESS` | Explicitly out of scope — the sponsor disclosed it |
| External references | `(Codex, PR #45)` | Fix landed via PR; check the PR actually covers the comment's claim |

Marker density is a prior: this codebase carried 13+ fixed IDs, 3 accepted-risk blocks, and a PR reference — the probability of a fresh Critical there is tail-end. Spend your first hour counting markers, not reading code.

## 2. Mirror-Consistency Checks (view vs. state-changing pairs)

Valuation systems pair a quoting function (view) with a settlement function (state-changing). The bug class is *formula drift between the pair*. Compare them on five axes, in a table:

1. How each input is computed (e.g. active-time accumulation)
2. The time window each uses (elapsed-since-checkpoint)
3. Clamps applied (`active <= elapsed`)
4. The core coefficient formula (fees × time × bps / denominator)
5. Which storage each reads/writes (pending escrow)

A drift on any axis = collateral is valued differently than it settles — over-borrowing or wrongful liquidation. In the audited codebase all five matched, with the quoting function simply not advancing checkpoints: an exact mirror. Closed.

## 3. Branch-Unreachability Proofs (div-by-zero candidates)

Every interpolated formula invites a "denominator can be zero/negative" candidate. The standard rejection path is not proving the denominator positive — it is proving **the parameter region where it isn't never intersects the branch entry condition**:

- Candidate: liquidation penalty interpolation divides by `startValue - maxPenaltyValue`, negative when `collateralFactor > 1/1.1`.
- Entry conditions: the branch requires `fullValue >= 1.1·debt`, and liquidation itself requires `collateral < debt`, which implies `fullValue < debt/factor`.
- Joint: `factor ≥ 0.909 ⇒ fullValue < 1.1·debt ⇒ branch unreachable`. `factor < 0.909 ⇒ denominator positive. ∎`

The invariant at the branch entrance (here: insolvency) is the bridge. No invariant, no proof — and then the candidate lives.

## 4. Rounding-Direction Lint (borrow/repay symmetry)

For every share/asset conversion, the rounding must favor the pool in *all four* directions: borrow-shares ceil, repay-by-asset shares floor, repay-by-share assets ceil, withdraw-shares floor. Check them as a set; a correct pair with a flipped sibling is the classic dust-leak.

## 5. The Honesty Rule for Audit Pace

Two rejected candidates on a heavily-marked codebase = the prior held. Continuing to grind the remaining modules "for completeness" spends time whose expected value is below an alternative target (another contest, another platform). The stopping rule: when your last two candidates died to their own invariants, and marker density said the tail was thin, close the audit and log what the pass *proved* — that a named codebase's core paths hold under your method. That proof is reusable; the third re-read of a clean module is not.

## Transfer

These five checks are portable to any contest on a mature codebase: count markers → mirror-table the quote/settle pairs → invariant-proof the interpolation candidates → lint the rounding set → apply the pace rule. The method's output is either a finding or a calibrated clean bill; both are worth exactly what the next decision needs.
