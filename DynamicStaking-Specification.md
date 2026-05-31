# DynamicStaking Specification

**Version:** 1.0
**Date:** 2026-05-30
**Authors:** FROGEN Team @ GenLabs Ltd.
**Status:** Canonical specification, subject to refinement through audit cycles.
**Repository:** https://github.com/frogen-protocol/frogen-specs
**License:** See repository [LICENSE](./LICENSE).

---

## 1. Overview

DynamicStaking is the staking subsystem of the FROGEN protocol. It accepts FROGEN
tokens from holders, locks them for one of six fixed terms between three and
twenty-four months, and pays a reward at maturity drawn from a separately tracked
reward inventory. Rates are determined at the moment of staking from on-chain pool
state and are frozen for the life of the position. There is no early withdrawal.

The mechanism is designed to address two coupled problems: producing predictable
holder economics in a volatile market, and reducing circulating supply by giving
long-horizon holders an explicit, transparent commitment vehicle. Reward solvency
is enforced by construction. The system maintains a hard accounting separation
between user principal and reward inventory, reserves every promised reward at
stake time, and refuses any stake whose reward would exceed available reward
inventory.

Within the wider FROGEN architecture, DynamicStaking sits alongside the
[FROGEN token contract](Cross-Contract-Specification.md#frogen-token), the
[SmartVest fee mechanism](SmartVest-Specification.md), and the
[MerkleClaim migration contract](Cross-Contract-Specification.md#merkleclaim). The token treats
the staking contract as a hard-coded system-exempt address, so deposits and
payouts bypass transfer fees. SmartVest, through its vault, periodically refills
the reward inventory from accumulated transfer fees. MerkleClaim brings pre-TGE
database-staked positions on-chain via a restricted migration path. The Presale
contract delivers any unsold-allocation tokens into the reward inventory after
sale completion.

---

## 2. Terminology and Notation

### 2.1 APR vs APY

> **Terminology note.** The names `baseAPY`, `finalAPY`, and `effectiveAPY` are
> used throughout this specification and in the contract source for continuity
> with established user-facing conventions. The reward math, however, is
> non-compounding simple interest:
>
> `promisedReward ≈ principal × rate × (months / 12)`
>
> This is the definition of **APR**, not APY. No automatic compounding occurs
> over a position's lifetime. The term `APY` is retained as a name, not as a
> mathematical claim. User-facing interfaces are required to display, before
> confirmation, the position-size-specific `effectiveAPY` and the absolute
> reward amount. If `baseAPY` or `finalAPY` is also displayed, it must be
> labelled as a headline/pre-weighting rate rather than the rate the user
> will necessarily receive.

### 2.2 Glossary

| Term | Definition |
|---|---|
| `principal` | The amount of FROGEN a user has locked in a single position. |
| `rewardInventory` | The staking contract's token balance minus `totalStakedPrincipal`. The pool of reward tokens. |
| `totalStakedPrincipal` | The sum of `principal` across all unclaimed on-chain positions, including matured-but-unclaimed positions. |
| `reservedRewards` | The sum of `promisedReward` across all unclaimed on-chain positions, including matured-but-unclaimed positions. |
| `migrationReservedRewards` | The sum of `promisedReward` earmarked for pre-TGE positions that have not yet been materialised on-chain. |
| `availableRewards` | `max(0, rewardInventory − reservedRewards − migrationReservedRewards)`. The pool depth available for new stakes. |
| `REFERENCE_POOL` | The fixed denominator used in the base APY formula. 155,400,000 × 10^18 tokens. |
| `baseAPY` | The current pool-determined APR before applying any lock-period multiplier. |
| `finalAPY` | `baseAPY` after applying the lock-period multiplier. The headline rate for a position's chosen lock term. |
| `effectiveAPY` | The actual rate a position locks in after the weighted reward formula is applied. May be less than `finalAPY` for large positions; never exceeds it. |
| `promisedReward` | The exact reward in tokens reserved at stake time and paid out at maturity. |
| `position` | A single staking record: wallet, principal, lock months, locked rate, promised reward, start timestamp, maturity timestamp, claimed flag. |
| `lockPercentageBps` | The fraction of circulating supply currently locked in the staking and vault subsystems, in basis points. Used by the fee mechanism, not by staking. |
| `poolHealthBps` | `floor(rewardInventory × 10000 / REFERENCE_POOL)`. The reward inventory as a fraction of the reference pool, in basis points. |

### 2.3 Notation

- Rates are represented in 18-decimal fixed point: `1e18` represents 100% per
  annum. `5e18` represents 500% per annum.
- Token amounts use the FROGEN token's 18-decimal precision.
- Basis-point (`Bps`) values are integers where 10000 represents 100%.
- All division is integer floor division. Saturating subtraction is used
  only for derived view quantities that explicitly define `max(0, ...)`,
  such as `availableRewards`. State-accounting decrements, including
  `totalStakedPrincipal`, `reservedRewards`, and
  `migrationReservedRewards`, use checked arithmetic and must revert on
  underflow.

---

## 3. Core Mechanism

### 3.1 Two-Bucket Accounting

The staking contract holds two categorically different kinds of tokens: user
principal and reward inventory. They are tracked separately, and the separation
is the foundation of solvency.

- `totalStakedPrincipal` records the sum of principal for all unclaimed
  positions, including active and matured-but-unclaimed positions. It is
  incremented on stake and migration, decremented on claim and on the closing
  half of `claimAndRestake`.
- `rewardInventory` is derived, not stored: it is the contract's own token
  balance minus `totalStakedPrincipal`. This derivation is authoritative;
  implementations must not maintain a separate counter that could drift from
  the token balance under direct transfers or edge-case behaviour.

Depositing principal must never be treated as a reward refill. Reward
inventory increases through authorised refill paths and direct donations,
decreases when rewards are paid, and becomes newly available when
migration reserves are released. Operations that move principal in or out
adjust `totalStakedPrincipal` in lockstep so that `rewardInventory`
remains an accurate measure of reward tokens.

### 3.2 Pools and Constants

| Constant | Value | Role |
|---|---|---|
| `REFERENCE_POOL` | 155,400,000 × 10^18 tokens | Denominator of the base APY formula and the pool-health metric. Not a hard cap on actual reward inventory. |
| `APY_MAX` | 5 × 10^18 (500% per annum) | Ceiling for `baseAPY` at any pool state. |
| `MIN_STAKE` | 1000 × 10^18 tokens | Minimum principal required for `stake` and for the non-zero re-stake portion of `claimAndRestake`. Pre-TGE migrated positions follow the Merkle-verified migration commitments and are not subject to `MIN_STAKE`. |
| `SECONDS_PER_MONTH` | 2,628,000 seconds | Fixed (365 days / 12). Maturity arithmetic is calendar-independent. |

`REFERENCE_POOL` is a denominator, not a cap. Actual `rewardInventory` may
exceed it through fee refills, treasury injections, presale-unsold transfer,
or direct donation. Excess inventory increases the system's runway and the
pool depth visible to large stakers; it does not increase any individual
position's `baseAPY` above `APY_MAX`.

### 3.3 Position Lifecycle

A position progresses through three observable states:

1. **Active.** Created by `stake`, `claimAndRestake`, or migration. The rate
   and promised reward are fixed at creation. The principal is locked.
2. **Matured.** `block.timestamp` has reached `maturityTimestamp`. The
   position becomes claimable. No state change occurs at maturity; the
   transition is purely temporal.
3. **Claimed.** The position has been paid out and is marked terminal.

The defining property of the lifecycle is **per-stake rate locking**: the
position's locked rate (`effectiveAPY`) and `promisedReward` determined
at creation are stored on the position and used at claim time without
re-evaluation against subsequent pool state. A stake created on day one at a
high rate is honoured at that high rate at maturity even if pool conditions
have changed substantially. A stake created when the pool is partially
depleted locks in the lower rate then prevailing for its entire duration.

There is no early withdrawal mechanism. Within the canonical implementation,
and short of a proxy upgrade, a position cannot be closed before its
`maturityTimestamp` by the user, by any role-bearing account, or by any
governance action. This is enforced both by the absence of such a function in
the public surface and by the construction of the reward accounting:
For an on-chain position, `promisedReward` is reserved from
`rewardInventory` at creation and released from `reservedRewards` only
when the position owner successfully closes the position through `claim`
or `claimAndRestake`; separate unmaterialised migration obligations are
released only through the migration-reserve release path described in
§10.6.

---

## 4. Lock Periods and Multipliers

A position must select one of exactly six lock terms. The set is fixed at
deployment and is not governance-tunable.

| Lock period (months) | Multiplier (decimal) | Multiplier (18-dec) | `finalAPY` at pool-saturated `baseAPY` = 500% | Implied total reward at maximum participation |
|---:|---:|---|---:|---|
| 3 | 0.15 | 1.5 × 10^17 | 75% APR | ~18.75% of principal |
| 6 | 0.30 | 3.0 × 10^17 | 150% APR | ~75% of principal |
| 9 | 0.45 | 4.5 × 10^17 | 225% APR | ~168.75% of principal |
| 12 | 0.60 | 6.0 × 10^17 | 300% APR | ~300% of principal |
| 18 | 0.80 | 8.0 × 10^17 | 400% APR | ~600% of principal |
| 24 | 1.00 | 1.0 × 10^18 | 500% APR | ~1000% of principal (10× principal) |

The "total reward at maximum participation" column reflects the unweighted
formula `principal × finalAPY × months / 12`, which is the ceiling on
promised reward for any stake (see §5.4 below). The weighted formula reduces
this ceiling for large positions in proportion to their pool impact.

The schedule of multipliers is strictly increasing in lock term. The
resulting `finalAPY` is weakly monotone after integer flooring, and is
strictly higher for longer locks except at zero or near-zero `baseAPY`
values where flooring can collapse adjacent outputs.

---

## 5. Mathematical Specification

### 5.1 Reward Inventory and Availability

```
rewardInventory = balanceStaking - totalStakedPrincipal
```

```
availableRewards = max( 0, rewardInventory - reservedRewards - migrationReservedRewards )
```

`availableRewards` is the pool depth visible to new stakes. It never reverts;
when the subtraction would underflow it clamps at zero.

### 5.2 Base APY

```
baseAPY = min( APY_MAX, floor( APY_MAX × availableRewards / REFERENCE_POOL ) )
```

`baseAPY` is linear in `availableRewards` until the pool reaches the reference
size, at which point it saturates at `APY_MAX`. The function is monotone
non-decreasing in `availableRewards`.

### 5.3 Final APY

```
finalAPY = floor( baseAPY × periodMultiplier[months] / 10^18 )
```

`finalAPY` is the headline rate associated with a lock term at current pool
state, before the weighted formula is applied.

### 5.4 Promised Reward (Weighted Formula with Cap)

Let `principal` be the actual amount of FROGEN received by the contract (see
§6 on the role of measured-after-transfer principal). Define an unweighted
candidate and a weighted candidate.

**Unweighted candidate** (the value the simple APR formula would produce):

```
R_unweighted = floor(  principal × finalAPY × months  /  (12 × 10^18)  )
```

**Weighted candidate** (asymptotically bounded by `availableRewards`):

```
             principal × finalAPY × months × availableRewards
R_weighted = ───────────────────────────────────────────────────────────────
             (12 × REFERENCE_POOL × 10^18) + (principal × finalAPY × months)
```

The actual promised reward is the smaller of the two:

```
promisedReward = min( R_weighted, R_unweighted )
```

Three properties of this construction are worth stating explicitly.

- **Small-stake limit.** As `principal × finalAPY × months` becomes small
  relative to `12 × REFERENCE_POOL × 10^18`, the weighted reward converges to
  `R_unweighted × availableRewards / REFERENCE_POOL`, before applying the
  unweighted cap. At a pool-saturated state where
  `availableRewards = REFERENCE_POOL`, this is approximately `R_unweighted`,
  so small stakers receive essentially the headline `finalAPY`. Below
  saturation, the effective rate is further reduced by the same
  `availableRewards / REFERENCE_POOL` factor.
- **Large-stake bound.** As `principal × finalAPY × months` grows large
  relative to the same denominator term, `R_weighted` approaches
  but never reaches `availableRewards`. A single stake can never consume the
  whole reward pool.
- **Cap.** The `min` against `R_unweighted` binds only when
  `availableRewards > REFERENCE_POOL` (an overfunded pool). It guarantees that
  effective APY never exceeds headline `finalAPY` even in a pool that has
  been replenished beyond the reference size.

### 5.5 Effective APY

The headline `finalAPY` reflects the rate before the weighted formula. The
rate the position actually locks in is:

```
effectiveAPY = (promisedReward × 12 × 10^18) / (principal × months)
```

This is the value the caller-supplied slippage minimum is compared against;
see §6.4. For small stakes it is approximately `finalAPY` only when the pool
is saturated or sufficiently overfunded for the unweighted cap to bind; below
saturation it is approximately `finalAPY × availableRewards / REFERENCE_POOL`.
For large stakes, it is less than `finalAPY` when the weighted formula is the
binding candidate; if the pool is sufficiently overfunded for the unweighted
cap to bind, `effectiveAPY` equals `finalAPY`.

### 5.6 Maturity Timestamp

```
maturityTimestamp = startTimestamp + (lockMonths × SECONDS_PER_MONTH)
```

For positions created via `stake` or `claimAndRestake`, `startTimestamp` is
`block.timestamp` at creation. For positions materialised via migration,
`startTimestamp` is the TGE timestamp regardless of when the migration call
lands; see §10.4.

### 5.7 Rounding and Integer Math

Every division in the mathematical specification is integer floor division.
Where two formulations of the same quantity differ by a rounding step, the
formulation that rounds in favour of protocol solvency (i.e. reduces the
promised reward by at most one unit) is the authoritative one. This is a
design choice in favour of reservoir solvency over arithmetic exactness, and
its worst-case effect on any user is sub-token rounding.

### 5.8 Solvency Invariants

Two invariants must hold at all times after every state-changing operation:

```
rewardInventory >= reservedRewards + migrationReservedRewards
```

```
balanceStaking >= totalStakedPrincipal + reservedRewards + migrationReservedRewards
```

The first invariant guarantees that every commitment recorded against the
reward pool is backed by reward tokens actually held by the contract. The
second guarantees that the contract collectively holds enough tokens to
return every active principal and pay every promised reward. The pre-transfer
APY-snapshot ordering (§6) and the snapshot-before-close ordering of
`claimAndRestake` (§8) ensure both invariants hold by construction; an
explicit post-operation check in the more complex paths catches any
implementation error.

### 5.9 Worked Examples

**Example A. Canonical 100,000-token, 24-month stake at maximum participation.**

Assume `availableRewards = REFERENCE_POOL`, so `baseAPY = APY_MAX = 500%`.
The 24-month multiplier is `1.00`, so `finalAPY = 500% × 1.00 = 500%`.

Step through the unweighted formula:

```
R_unweighted = (100,000 × 5.00 × 24) / 12 = 1,000,000 FROGEN
```

At exactly `availableRewards = REFERENCE_POOL`, the weighted formula returns
approximately 993,606 FROGEN. This is close to, but below, the unweighted
1,000,000 FROGEN ceiling, so the cap does not bind. At maturity the staker
receives approximately 1,093,606 FROGEN total: 100,000 FROGEN principal plus
approximately 993,606 FROGEN reward.

**Example B. Concentration penalty at 15.54M-token, 24-month stake at
maximum participation.**

Same pool conditions: `baseAPY = 500%`, `finalAPY = 500%` for the 24-month
lock. The unweighted formula gives:

```
R_unweighted = (15,540,000 × 5.00 × 24) / 12 = 155,400,000 FROGEN
```

This is exactly the entire reference pool. Under the unweighted formula a
single stake of this size would attempt to drain the whole reward pool.
The weighted formula produces instead approximately 77,700,000 FROGEN, and
the cap does not bind, so `promisedReward ≈ 77,700,000 FROGEN`. The
effective APY locked in is approximately 250%, not 500%. The remaining
~77.7M tokens in the reward pool are preserved for subsequent stakers. This
example illustrates the weighted formula's role: it makes large stakes
proportionally less efficient at consuming the pool, ensuring that no single
participant can foreclose participation for others while remaining within
the reservation accounting.

---

## 6. Staking Operation

### 6.1 Pre-conditions

A `stake` call must satisfy all of the following:

- The contract is not paused.
- The TGE timestamp has been set and `block.timestamp` is at or after it.
- The migration snapshot has been finalised.
- The requested lock period is exactly one of {3, 6, 9, 12, 18, 24}.
- The requested principal is at least `MIN_STAKE`.

Any unmet pre-condition reverts before any token movement.

### 6.2 Operational Sequence

The stake operation proceeds in a fixed order. Several of these steps are not
merely implementation choices but mechanism-level requirements; deviations
would compromise solvency or allow self-dealing.

1. **Snapshot before any token movement.** Compute `availableRewards`,
   `baseAPY`, and `finalAPY` from the contract's state as it stands before
   any principal is transferred in.
2. **Pull principal with balance-before/after measurement.** The actual
   principal credited to the position is the post-transfer balance minus the
   pre-transfer balance, not the requested amount. This makes the contract
   robust to any transfer-fee or non-standard token behaviour and ensures
   that `totalStakedPrincipal` reflects what the contract actually received.
3. **Re-check minimum after measurement.** Even though the requested amount
   was validated against `MIN_STAKE`, the actually-received amount is
   re-checked, in case any deduction reduced it below the threshold.
4. **Compute `promisedReward` from the snapshot.** Both candidates in §5.4
   are computed using the pre-transfer `availableRewards`, not any
   post-transfer recomputation. `promisedReward` is the minimum of the two.
5. **Verify effective APY.** Compute `effectiveAPY` per §5.5 and require it
   to be at least the caller's `minAcceptableAPY`. This is the slippage check.
6. **Verify the reservation fits.** Require `promisedReward ≤ availableRewards`
   (the pre-transfer snapshot). The weighted formula and its cap guarantee
   this mathematically; the explicit check provides defence in depth against
   any future change to the formula and surfaces invariant violations loudly
   during audit.
7. **Verify non-zero reward.** Require `promisedReward > 0`. If pool
   depletion has reduced `finalAPY` to a value at which the reward formula
   rounds to zero, the stake reverts.
8. **Update accounting and create position.** Increment
   `totalStakedPrincipal` and `reservedRewards`, record the position
   (wallet, principal, lock months, `effectiveAPY` as the locked rate,
   `promisedReward`, start and maturity timestamps), and emit the
   creation event.

### 6.3 Why the Pre-Transfer Snapshot Is Required

The base-APY formula uses `availableRewards`, which depends on
`rewardInventory = balance − totalStakedPrincipal`. If `availableRewards`
were recomputed after the principal transfer but before
`totalStakedPrincipal` is incremented, the just-deposited principal would
appear inside `rewardInventory` and would inflate the staker's own
`baseAPY`. Under the weighted formula the effect would be amplified: a
large stake could give itself a self-financed APY boost, drain the reward
pool, and potentially violate the solvency invariant after the transfer.

The required ordering (snapshot before transfer, formula uses the
snapshot, accounting updated after) is a key invariant of the mechanism.

### 6.4 Slippage Protection

The caller supplies `minAcceptableAPY` as a floor. The contract rejects the
stake if `effectiveAPY < minAcceptableAPY`. The comparison is against the
post-weighting effective rate, not the headline `finalAPY`, so the
protection accounts for the user's own pool impact. A user who supplies
`minAcceptableAPY = 0` accepts whatever effective rate the pool offers,
but the stake still reverts if `promisedReward` rounds to zero (see §6.2
step 7).

**Transaction-ordering and MEV note.** `minAcceptableAPY` is revert-only
protection; it does not reserve a quoted rate or guarantee access to an
APY displayed off-chain. Any earlier transaction in execution order,
including stakes submitted after a visible refill or direct donation,
can reduce `availableRewards` and therefore reduce the caller's
`effectiveAPY`. Users and integrations must treat displayed APY as an
estimate until the transaction executes, and should set
`minAcceptableAPY` accordingly.

---

## 7. Claim Operation

### 7.1 Pre-conditions

A `claim` call must satisfy:

- The position exists.
- The caller is the position's recorded wallet.
- `block.timestamp ≥ maturityTimestamp`.
- The position has not already been claimed.

### 7.2 Effects

The position is marked claimed. `totalStakedPrincipal` is decremented by the
position's principal. `reservedRewards` is decremented by the position's
`promisedReward`. The contract transfers `principal + promisedReward` to the
position's wallet in a single transfer. The accounting updates and the
transfer are atomic; a failure in either reverts the whole operation.

### 7.3 Claim Is Not Blocked by Pause or Migration State

`claim` is not gated by the pause state, and it is not gated by
migration-snapshot finalisation. Under the canonical implementation, once
a position has matured it is claimable regardless of pause state or
migration-snapshot finalisation. The reserved tokens backing a matured
position were earmarked at stake time and have no dependency on later
launch-accounting state. This is a deliberate "no funds held hostage"
property of the canonical implementation: governance operations that
pause new staking or that touch migration accounting cannot prevent a
matured position from being claimed, subject to the upgrade-path trust
assumption described in §14.4 and §16.4.

---

## 8. Claim-and-Restake Operation

### 8.1 Purpose

`claimAndRestake` is a single atomic operation that closes a matured position
and optionally creates a new position funded from part or all of the closing
payout, with the re-staked portion never leaving the staking contract; any
non-restaked remainder is transferred to the user fee-free. Because the
staking contract is a system-exempt address from the token's perspective, no
transfer fee is incurred on any path through this operation, including the
withdrawal portion when only part of the payout is restaked.

### 8.2 Operational Sequence

The operation has a required two-phase ordering: snapshot first, then close
old, then open new.

1. **Validate closing position and restake parameters.** Require that the
   position exists, the caller is the position's recorded wallet,
   `block.timestamp ≥ maturityTimestamp`, and the position has not already
   been claimed. Compute `totalPayout = principal + promisedReward`.
   Require `restakeAmount ≤ totalPayout`. If `restakeAmount > 0`, require
   `restakeAmount ≥ MIN_STAKE`, require `newLockMonths` to be one of
   `{3, 6, 9, 12, 18, 24}`, require the TGE timestamp has been set and
   `block.timestamp ≥ tgeTimestamp`, and require the migration snapshot
   has been finalised. When `restakeAmount = 0`, `newLockMonths`,
   `minAcceptableAPY`, and the TGE/snapshot gates are ignored, preserving
   zero-restake equivalence to `claim`.
2. **Snapshot before closing.** If `restakeAmount > 0`, capture the current
   `availableRewards`, `baseAPY`, and `finalAPY` for the new lock period
   from the state as it stands with the old position still active.
3. **Close the old position.** Mark it claimed; decrement
   `totalStakedPrincipal` by its principal; decrement `reservedRewards` by
   its `promisedReward`.
4. **Open the new position.** If `restakeAmount > 0`, compute
   `promisedReward` for the new position using the weighted formula with the
   snapshotted values; validate slippage, require `promisedReward > 0`, and
   require `promisedReward ≤ snapshotted availableRewards`; then increment
   `totalStakedPrincipal` by `restakeAmount`, increment `reservedRewards` by
   the new `promisedReward`, and record the position.
5. **Withdraw remainder.** If `restakeAmount < totalPayout`, transfer
   `totalPayout − restakeAmount` to the caller. The transfer is fee-free.
6. **Solvency check.** Verify the solvency invariants hold after the
   combined operation.

### 8.3 Why Snapshot Before Close

If the new position's `availableRewards` were computed after closing the old
one, it would be inflated by the released `principal + promisedReward`. A
large new position could promise rewards backed in part by tokens about to
be paid out as the withdrawal portion, violating the solvency invariant.
Snapshotting before close guarantees that the new position's reservation is
bounded by `availableRewards` as it stood with the old reservation still in
place. The withdrawal portion of the operation cannot reduce reservation
capacity for the new position.

### 8.4 Pause Exemption

`claimAndRestake` is exempt from the pause state. The rationale is that the
operation does not introduce new external capital into the contract; it
recycles tokens that the contract already holds. The pause mechanism is
designed to halt fresh external deposits via `stake`, not internal
recycling. Allowing `claimAndRestake` during a pause means a user whose
position matures during an incident-response pause is not forced into a
withdrawal-only path.

### 8.5 Parameter Semantics

- `restakeAmount`: either zero, producing a pure claim, or a value in
  `[MIN_STAKE, totalPayout]`. Values in `(0, MIN_STAKE)` revert.
- `newLockMonths`: validated against the lock-period allowlist only when
  `restakeAmount > 0`. Ignored otherwise.
- `minAcceptableAPY`: the slippage floor for the new position's
  `effectiveAPY`. Compared against the same effective rate definition used
  in `stake` (§5.5).

### 8.6 Zero-Restake Equivalence

When `restakeAmount = 0`, the operation is functionally equivalent to
`claim`. Like `claim`, this form is not gated on migration-snapshot
finalisation and not blocked by pause. This preserves the "no funds held
hostage" property: a position whose owner wishes only to take payout cannot
be blocked by any governance state.

### 8.7 New Position Uses the Same APY Math

A position opened via `claimAndRestake` uses the same weighted formula and
the same cap as a position opened via `stake`. There is no preferential
rate for restakers.

---

## 9. Pool Depletion Behaviour

The reward pool can be depleted by accumulated stakes during periods when
refills lag demand. The mechanism's response is to prefer existing
commitments over new ones, deterministically.

- **Active positions are honoured to maturity under the canonical
  implementation.** Every active position's
  `promisedReward` was reserved from `rewardInventory` at stake time and is
  carried as part of `reservedRewards` until claimed. Under the canonical
  implementation, and short of a proxy upgrade, reservations cannot be
  cancelled, reduced, or reassigned by any role or process. Depletion
  affects only the visibility of new staking, not the redemption of
  existing positions.
- **New stakes revert when reward rounds to zero.** As `availableRewards`
  approaches zero, `baseAPY` and therefore `finalAPY` approach zero. When
  the resulting `promisedReward` rounds to zero, `stake` reverts. A user
  cannot lock principal for no reward, even by setting `minAcceptableAPY =
  0`.
- **New stakes revert when slippage protection binds.** If the
  caller-supplied `minAcceptableAPY` exceeds the current `effectiveAPY`,
  `stake` reverts before any state change.
- **Recovery occurs through refills.** The reward pool is restored when tokens
  enter through the refill paths described in §11. As `availableRewards`
  rises, `baseAPY` rises with it, and new stakes again become possible.

Under the canonical implementation and absent a proxy upgrade, this
combination prevents pool depletion from jeopardising existing locks,
while still permitting the staking subsystem to refuse new commitments
it cannot honour.

---

## 10. Pre-TGE Position Migration

### 10.1 Purpose and Scope

A portion of FROGEN's pre-launch presale period involved off-chain database
staking commitments. Those commitments are materialised on-chain after TGE
via a Merkle-gated migration path. The result is an on-chain position with
the same lock term, the same locked rate, and the same maturity schedule
the user agreed to off-chain, regardless of when the migration call is made
within the migration window.

### 10.2 Two-Phase Launch Accounting

Migration introduces pre-committed reward obligations that must be
accounted for before any general staking activity is permitted. The
mechanism handles this in two phases.

**Phase 1: TGE timestamp set.** A timelocked role sets the TGE timestamp
once, post-deployment, to a value at least seven days in the future. Until
this is done, the TGE timestamp is zero and every TGE-dependent operation
(`stake`, `migrateStakeFor`, restaking through `claimAndRestake`) reverts.
Setting the TGE timestamp also derives the migration window end
(TGE + 12 months) as a side effect.

**Phase 2: Migration snapshot finalised.** A timelocked role records
`migrationReservedRewards` as the sum of `promisedReward` across all
database-staking commitments being migrated, in a single call. This must
occur before any user action that touches launch accounting. The
finalisation step also performs a solvency check: it reverts if
`rewardInventory < reservedRewards + migrationReservedRewards`, ensuring
the contract holds enough reward tokens to honour all pre-committed
positions before users can interact.

After the first liability-inflating user action (a `stake`, a
`claimAndRestake` with non-zero restake, or a `migrateStakeFor` call),
launch accounting is sealed. The finalisation function refuses further
calls. This is a one-way transition. Because snapshot finalisation is a
one-shot operation, any governance correction to the migration reserve
amount must occur before finalisation; after finalisation, correction
requires a contract upgrade.

### 10.3 Migration Window

The migration window opens at the TGE timestamp and closes at TGE + 12
months. The window may be extended by a timelocked role to a later
timestamp. It cannot be shortened. While `block.timestamp >
migrationEndTimestamp`, the migration entry point reverts. If
`migrationReservedRewards` is still non-zero, a timelocked extension can
set `migrationEndTimestamp` to a future timestamp, and migration calls
are accepted again until the extended window closes. Once
`migrationReservedRewards` reaches zero, whether through a successful
release or through accumulated user migrations exhausting the reserve,
extension is structurally blocked: the `extendMigrationEndTimestamp`
entry point reverts when `migrationReservedRewards == 0`. After this
point, released or exhausted migration obligations cannot be recreated
absent a contract upgrade.

### 10.4 Per-Position Behaviour

A migrated position carries the exact `apyLocked` value the user agreed to
off-chain, as verified by the Merkle proof. The staking contract does not
recompute the rate. It accepts the Merkle-verified `apyLocked` value,
measures the actual principal received using balance-before/after
accounting, and derives
`promisedReward = floor(actualPrincipal × apyLocked × lockMonths / (12 × 10^18))`.
The recorded position principal and the `totalStakedPrincipal` accounting
both use `actualPrincipal` rather than the Merkle-authorised principal
amount. Under the canonical FROGEN configuration, where DynamicStaking is
a hard-coded system-exempt address and FROGEN is fixed-supply,
`actualPrincipal` equals the Merkle-authorised principal. The
balance-before/after pattern is defensive against any future
fee-on-transfer behaviour that would deduct from the principal in transit.

`MIN_STAKE` does not apply to migrated positions; the principal commitment
is whatever was authorised in the Merkle leaf at proof construction.
The `startTimestamp` is the TGE timestamp, not the timestamp of the
migration call. The `maturityTimestamp` is therefore
`TGE + lockMonths × SECONDS_PER_MONTH`.

This timing rule has two consequences. First, the lock schedule the user
agreed to in the pre-TGE period is honoured exactly, regardless of when
the user claims their migration. Second, a position whose lock term has
already elapsed by the time the user submits the migration call is
materialised as already-matured.

### 10.5 Late-Claim Behaviour

A position materialised as already-matured is immediately claimable. The
user can call `claim` (or `claimAndRestake`) on it without further wait.
This is the intended behaviour for users who submit their migration after
their lock term has run.

### 10.6 Migration Reserve Release

`migrationReservedRewards` is initially the sum of obligations for every
pre-TGE position. As users migrate, each `migrateStakeFor` call subtracts
the materialised obligation from `migrationReservedRewards` and adds it to
`reservedRewards`. Any remaining balance at the end of the migration
window represents pre-TGE positions whose holders did not claim.

A timelocked release path exists for this residual, with two enforced
delays:

1. The release cannot occur earlier than three months after
   `migrationEndTimestamp`. This grace period prevents immediate release after
   the migration window closes and gives governance time to extend the
   migration window if a late-claimant issue is identified.
2. A 14-day public on-chain notice must be posted before the release call
   will succeed. The 14-day timer is restarted on every subsequent notice
   posting. Both the notice posting and the eventual release are gated on
   a non-zero `migrationReservedRewards`. If `migrationEndTimestamp` is
   extended after a release notice has been posted, the existing notice is
   cleared and a fresh 14-day notice must be posted after the extended
   migration window has ended before migration reserves can be released.

A successful release zeroes `migrationReservedRewards`. The previously
earmarked tokens are not transferred anywhere: they remain in the
contract's `rewardInventory` and become part of the general
`availableRewards` pool. They are then visible to new stakers and
contribute to higher `baseAPY` for subsequent stakes.

### 10.7 Snapshot Gating: What Is and Is Not Blocked

The migration-snapshot mechanism gates only those operations that would
inflate reward liabilities or rely on the snapshot being meaningful:

| Operation | Gated on snapshot finalised? |
|---|---|
| `stake` | Yes. Reverts until snapshot is finalised. |
| `claimAndRestake` with `restakeAmount > 0` | Yes. The re-stake portion would create a new liability. |
| `migrateStakeFor` | Yes. The migration accounting move requires the snapshot to exist. |
| `claim` | No. Matured positions are claimable unconditionally. |
| `claimAndRestake` with `restakeAmount = 0` | No. Pure claim path; equivalent to `claim`. |
| `unpause` | Yes, with additional solvency check. New staking cannot resume against an unfinalised or under-funded reserve state. |

The asymmetry is intentional: launch accounting prevents the system from
being put into an inconsistent state by inflating liabilities, while
matured payouts are not blocked by pause state or migration-snapshot
state under the canonical implementation.

---

## 11. Reward Pool Refill Paths

The reward pool is replenished through three authorised entry points that use
restricted callers and balance-before/after measurement, plus direct token
transfers that are treated as donations to reward inventory.

### 11.1 SmartVest Fee Refill

The SmartVest vault is the regular refill source. Accumulated transfer
fees in the vault are periodically distributed; the staking subsystem's
share is delivered through this path. Only the SmartVestVault address may
call it. The transfer uses a pull pattern (the vault pre-approves; the
staking contract pulls), and the actual credited amount is measured as
`balance_after − balance_before`, not the requested amount. Calls with
zero amount are accepted as silent no-ops so that the vault can issue
optimistic calls without race-condition concerns. Calls with non-zero
amount that result in zero received tokens revert, surfacing any
misconfiguration loudly rather than allowing the vault's accounting to
desynchronise from staking's.

### 11.2 Presale Unsold-Allocation Refill

If any portion of the presale allocation goes unsold, the Presale contract
delivers the residual to the staking reward pool as a one-shot refill.
Only the Presale address may call this path. The amount parameter must
be non-zero; the actual credited amount is measured via balance-before/after.
A zero-received outcome reverts. This protects the Presale's internal
one-shot `unsoldTransferred` flag from being raised on a silent failure.

### 11.3 Emergency Refill

An emergency refill path is available to holders of the emergency role.
Any such caller can pre-approve the staking contract for an amount and
then trigger the refill; the contract performs balance-before/after
measurement to credit the actual received amount. The `amount` parameter
must be non-zero, and a non-zero requested amount that results in zero
actual tokens received reverts. The credited refill amount is always the
measured `balance_after − balance_before`, not the requested amount.
There is no hard-coded treasury source; the source is whichever account
holds the role and pre-approves the transfer. This path exists for
incident response when the regular refill channels are insufficient or
temporarily unavailable.

### 11.4 Direct Transfers

FROGEN tokens transferred directly to the staking contract address outside
the canonical refill paths are treated as donations to the reward pool.
They are reflected in subsequent `rewardInventory` computations
automatically (since `rewardInventory` is derived from balance) and are
not refundable. The contract cannot distinguish a deliberate donation
from an accidental transfer, and the protocol does not provide a
recovery mechanism for tokens sent in error.

---

## 12. Integration Points

This section describes how DynamicStaking interacts behaviourally with
other FROGEN contracts. Implementation-level call surfaces are not in
scope; the corresponding specifications govern those.

### 12.1 FROGEN Token

The staking contract is a hard-coded system-exempt address in the FROGEN
token. Transfers from the token contract perspective both into and out of
the staking contract bypass the fee logic entirely. This is what enables
fee-free stake deposits, fee-free claim payouts, fee-free refill
deliveries, and fee-free `claimAndRestake` operations.

Three other contracts share this hard-coded system-exempt status in the
FROGEN token contract: SmartVestVault, MerkleClaim, and PrizeTimeLock.
The complete list of four is fixed at FROGEN token deployment and cannot
be modified.

The token reads `totalStakedPrincipal` from the staking contract and reads
the staking contract's token balance from its own balance mapping. The
staking balance and `totalStakedPrincipal` are used to derive `poolHealthBps`.
The fee-curve `lockPercentageBps` is derived from locked principal and the
SmartVest vault balance according to the FROGEN token and SmartVest
specifications; staking reward inventory is excluded from `lockPercentageBps`.
See the
[Cross-Contract Specification](Cross-Contract-Specification.md#frogen-token)
for the FROGEN token section. If the `totalStakedPrincipal` call fails
for any reason, the token fails closed: `poolHealthBps` is set to zero
(maximum floor activation), forcing the core fee to `maxCoreFeeBps`; the base-fee component follows the token-side base-fee rules in the SmartVest specification. This is
conservative behaviour designed to preserve the refill mechanism's
safety properties under any failure condition.

### 12.2 SmartVest and SmartVestVault

The staking contract does not compute fees. It exposes the state the fee
curve and pool-health floor read from. The fee mechanism's curve, floor,
and parameter governance are documented in the
[SmartVest Specification](SmartVest-Specification.md).

The SmartVestVault is the only authorised caller of the regular
fee-refill path (§11.1). The staking contract verifies the caller
identity directly by address comparison; no role grant is involved.

### 12.3 MerkleClaim

The MerkleClaim contract is the only authorised caller of
`migrateStakeFor`. The migration entry point verifies caller identity by
address comparison. MerkleClaim is responsible for proof verification,
replay protection on the user's claim, and approving the staking contract
for the principal transfer; the staking contract is responsible for the
window check, the balance-before/after measurement of received principal,
the migration-reserve accounting move, and the position creation. See the
[Cross-Contract Specification](Cross-Contract-Specification.md#merkleclaim)
for the MerkleClaim section detailing proof construction and leaf encoding.

### 12.4 Presale

The Presale contract is the only authorised caller of the
unsold-allocation refill path (§11.2). Caller identity is verified by
direct address comparison. See the
[Cross-Contract Specification](Cross-Contract-Specification.md#presale)
for the Presale section covering the finalisation flow that triggers the
refill.

---

## 13. Edge Cases

| Case | Behaviour |
|---|---|
| First staker on a full pool | `baseAPY = APY_MAX = 500%`. `finalAPY` is `APY_MAX × multiplier`. The weighted formula is close to the unweighted ceiling only for sufficiently small positions; larger positions receive lower `effectiveAPY` according to §5.4 and §5.5. |
| Staker at exactly remaining capacity | The stake succeeds if `promisedReward > 0`, slippage passes, and the explicit `promisedReward ≤ availableRewards` reservation check passes. Under the current weighted formula, `R_weighted < availableRewards` whenever `availableRewards > 0` and `principal × finalAPY × months > 0`; the unweighted cap can only reduce the result further. |
| Pool overfunded (`availableRewards > REFERENCE_POOL`) | `baseAPY` saturates at `APY_MAX`; `finalAPY` is capped accordingly. The unweighted-formula cap binds for any stake whose weighted reward would otherwise exceed it, ensuring `effectiveAPY ≤ finalAPY`. |
| Reward pool depleted (`finalAPY` rounds to zero) | `stake` reverts with a non-zero-reward check. Existing positions are unaffected. `claim` continues to work on any matured position. |
| Below-`MIN_STAKE` principal | Rejected before any token movement; also rejected after measurement if a transfer-fee deduction reduces actual principal below the threshold. |
| Stake attempted before TGE timestamp set | Reverts. The same applies to `migrateStakeFor` and to `claimAndRestake` when `restakeAmount > 0`. |
| Stake attempted before migration snapshot finalised | Reverts. The same applies to `migrateStakeFor` and to `claimAndRestake` when `restakeAmount > 0`. `claim` and `claimAndRestake(0)` proceed. |
| Stake attempted while paused | Reverts. `claim`, `claimAndRestake`, refills, and migrations proceed. |
| Multiple positions per user | Each position is independent. Maturities, locked rates, and reward reservations do not interact across a user's positions. |
| Re-entry attempt during token transfer | All token-touching state-changing entrypoints, including `stake`, `claim`, `claimAndRestake`, `migrateStakeFor`, and refill paths, are non-reentrant; any nested call reverts before observing intermediate accounting state. |
| Pause state matrix | `stake` blocked; `claim` and `claimAndRestake` (both forms) allowed; `refillFromSmartVest`, `refillFromPresale`, `emergencyRefill`, and `migrateStakeFor` allowed. |

---

## 14. Governance and Access Control

### 14.1 Role Overview

Three categories of privileged caller exist:

- **Timelocked role.** Sets the TGE timestamp, finalises the migration
  snapshot, unpauses the contract, extends the migration window, posts the
  release notice, and releases the migration reserves. At launch, this
  role is intended to be held only by the TimelockController, so calls
  routed through it are delayed by the TimelockController's configured
  delay, 48 hours at launch. If governance later grants this role directly
  to another account, that account can invoke the role-gated functions
  without a per-call TimelockController delay; such a grant should be
  treated as delegating the corresponding authority after the grant
  proposal's own timelock delay.
- **Emergency role.** Pauses the contract (immediate, no timelock); calls
  `emergencyRefill`.
- **Restricted-address callers.** Three specific contract addresses are
  hard-wired as the sole authorised callers of three specific entry
  points: the SmartVestVault for `refillFromSmartVest`, the MerkleClaim
  contract for `migrateStakeFor`, and the Presale contract for
  `refillFromPresale`. These addresses are set once at initialisation and
  cannot be changed.

### 14.2 Capabilities

The complete list of governance-callable operations on the DynamicStaking
implementation, excluding proxy-admin upgrades and TimelockController
administration covered in §14.4, is:

- Pause (emergency role; immediate).
- Unpause (timelocked role; gated on solvency and migration-snapshot
  finalisation).
- Set TGE timestamp (timelocked role; once; minimum 7 days in the
  future).
- Finalise migration snapshot (timelocked role; once; before any
  liability-inflating user action).
- Extend migration window (timelocked role; only later, never earlier).
- Post migration-reserve release notice (timelocked role; after migration
  window has ended; restarts the 14-day timer on each call).
- Release migration reserves (timelocked role; after the 3-month grace
  and 14-day notice have both elapsed).
- Emergency refill (emergency role; pulls approved tokens from the
  emergency-role caller into the reward pool).

Proxy upgrades and TimelockController delay changes are
governance-controlled surfaces but are not staking implementation
entrypoints; they are covered in §14.4 and §15.

Role membership administration is a governance-controlled surface. The
TimelockController, as role administrator, may grant or revoke the
timelocked and emergency roles through the standard access-control
role-admin paths, subject to the 48-hour timelock delay. Role-admin
operations do not directly modify positions, reserved rewards, migration
reserves, or user funds, but they can change which accounts may invoke
the role-gated operations listed above. Role holders may also renounce
their own roles. There is no on-chain cap on the number of role holders.

### 14.3 Limitations

The following capabilities are explicitly absent from the mechanism. They
cannot be exercised by any role, any combination of roles, or any
on-chain action short of a contract upgrade:

- Modify a position's recorded principal, locked rate, lock term, or
  maturity timestamp after creation, or modify a position's claimed
  flag except through the position owner's successful claim or
  claimAndRestake path.
- Withdraw a user's principal or any reserved reward from the contract by
  any path other than the position owner's own `claim` or
  `claimAndRestake` call.
- Cancel, edit, or remove a position.
- Reduce or reassign `reservedRewards` or `migrationReservedRewards` outside
  the normal accounting paths described in this specification: position-owner
  `claim` or `claimAndRestake`, authorised `migrateStakeFor` materialisation,
  and the migration-reserve release path (§10.6).
- Block or delay a matured position's claim. The pause mechanism applies
  to `stake` only.

### 14.4 Upgrade Path

The staking contract is deployed behind a transparent upgradeable proxy.
The proxy admin is owned by a TimelockController configured with a 48-hour
delay at launch. The governance multisig is the proposer and executor on the
TimelockController. An upgrade therefore requires both the multisig's
approval and the elapse of the timelock delay before execution.

This upgrade path is the principal residual trust assumption of the
mechanism. Within the canonical implementation no role can compromise an
active position's terms, but a contract upgrade could in principle deploy
an implementation that changes behaviour. The 48-hour delay provides a
public review window during which users can observe the proposal and
elect not to create new positions. Existing stakers cannot withdraw early
in response to an upgrade proposal; for already-locked positions, the
timelock provides notice and review time but not an on-chain exit right.
The TimelockController's delay can itself be modified via the same
schedule-and-execute flow, which means any proposal to reduce the delay
would itself be subject to the current delay (at launch, 48 hours).
Permanently enforcing a minimum delay would require a TimelockController
subclass, which is not deployed in the canonical Phase 1 implementation.
An optional eventual upgrade-freeze policy (permanently disabling
upgrades or extending the delay further) is a governance choice that may
be adopted after the system has accumulated sufficient operational
history.

---

## 15. Parameter Reference

| Parameter | Type | Launch value | Governable range | Constraint |
|---|---|---|---|---|
| `REFERENCE_POOL` | uint256 | 155,400,000 × 10^18 | Immutable | Compile-time constant. Denominator of `baseAPY` and `poolHealthBps`. Must equal the reference pool value used by the FROGEN token. |
| `APY_MAX` | uint256 | 5 × 10^18 (500%) | Immutable | Compile-time constant. Ceiling on `baseAPY`. |
| `MIN_STAKE` | uint256 | 1000 × 10^18 (1000 tokens) | Immutable | Compile-time constant. Floor on principal for `stake` and for the re-stake portion of `claimAndRestake`. |
| `SECONDS_PER_MONTH` | uint256 | 2,628,000 | Immutable | Compile-time constant. Fixed (365 days / 12). |
| Lock period set | set of uint256 | {3, 6, 9, 12, 18, 24} | Immutable | Validated as an explicit allowlist. |
| Period multipliers | mapping uint256 → uint256 | 0.15 / 0.30 / 0.45 / 0.60 / 0.80 / 1.00 (18-dec) | Immutable | Fixed schedule at deployment; not governance-tunable. |
| TGE minimum future offset | uint256 | 7 days | Immutable | Minimum delta between `block.timestamp` and the value of the TGE timestamp at the time it is set. |
| Migration window length | uint256 | 12 × `SECONDS_PER_MONTH` | Extendable (timelocked role) | Initial window = TGE + 12 months. May be extended to any later timestamp; may never be shortened. |
| Migration release grace period | uint256 | 3 × `SECONDS_PER_MONTH` | Immutable | Minimum delay between `migrationEndTimestamp` and the earliest possible execution of the migration-reserve release. |
| Migration release notice delay | uint256 | 14 days | Immutable | Minimum delay between the most recent notice posting and a successful release call. |
| Upgrade timelock delay | uint256 | 48 hours | Settable on the TimelockController per its own governance process; not enforced as a hard minimum in the canonical implementation. | Applies to staking calls routed through the TimelockController-held role. It does not automatically apply to direct role holders if governance grants `TIMELOCK_ROLE` to an address other than the TimelockController. |
| `maxCoreFeeBps` (integration) | uint256 | 650 bps (6.50%) | 100–850 bps via timelocked governance | Set on SmartVest, not on staking. Affects refill rate via the fee curve. See [SmartVest Specification](SmartVest-Specification.md). |
| `CAP_FLOOR_BPS` (integration) | uint256 | 25 bps | Immutable | Curve-wide minimum core fee. Set on SmartVest. |
| Pool-health floor threshold (integration) | uint256 | 3000 bps (30%) | Immutable | `poolHealthBps` value below which the floor activates. Set on SmartVest. |

The integration parameters (`maxCoreFeeBps`, `CAP_FLOOR_BPS`, pool-health
floor threshold) are listed here because they affect refill behaviour
visible to staking. They are governed by the SmartVest contract, not by
the staking contract. Other initial parameter values for which final
deployment configuration is pending are listed as "TBD at deployment" in
the corresponding cross-contract specifications.

---

## 16. Design Trade-offs

### 16.1 Sequential Staking Marginal Advantage

The weighted formula is concave with respect to position size at a fixed
pool snapshot. For actual sequential stakes that recompute
`availableRewards` after each transaction, the formula is approximately
path-independent when `availableRewards ≤ REFERENCE_POOL`; splitting a
stake does not materially improve total rewards beyond floor-rounding
effects. In overfunded states where `baseAPY` is saturated and the
unweighted cap may bind, splitting can produce an economically meaningful
advantage depending on overfunding level, total stake size, and split
granularity. The protocol does not aggregate same-wallet stakes, so each
transaction is evaluated independently against the then-current pool state.

### 16.2 No Early Exit

Principal is illiquid for the duration of a position's lock. This is a
deliberate design choice rather than a limitation: the mechanism's value
to the protocol (predictable supply lock and aligned holder commitment)
depends on the credibility of the no-early-exit rule. Users who require
liquidity flexibility should not stake.

### 16.3 Maturity Concentration / Unlock Cliff

Positions mature at deterministic timestamps and, under the canonical
implementation, a successful claim returns principal plus the full
promised reward in a single transaction. If many positions are created
in the same time window with the same lock term, a large amount of
principal and rewards can become claimable in the same later window. The
protocol does not stagger, throttle, or delay matured claims. This can
increase circulating supply and potential sell pressure around common
maturity dates. UI and analytics should surface aggregate upcoming
maturities by time bucket. Any staggered-release alternative would
require a future optional mechanism or upgrade and must not alter
existing position terms.

### 16.4 Upgrade-Path Trust Assumption

The transparent proxy architecture introduces a residual trust assumption
that no role-internal capability can match: governance could, in
principle, deploy a new implementation that changes behaviour. The 48-hour
timelock and the public review window mitigate but do not eliminate this.
This is particularly relevant for existing stakers: because there is no
early withdrawal, the timelock window is observational, not an escape
route. Existing locked positions cannot exit in response to an upgrade
proposal. Under the canonical implementation their recorded terms remain
unchanged, but an executed upgrade could change the logic applied to
existing positions; the timelock provides notice and review time, not an
on-chain guarantee that prior terms remain enforceable.
An eventual upgrade-freeze adoption (extending the delay further,
deploying a TimelockController subclass that enforces a minimum delay, or
permanently disabling upgrades) is a governance choice available to the
protocol after operational history accumulates.

### 16.5 APY Terminology

The rates exposed under the `baseAPY` and `finalAPY` names are APR-style
simple-interest rates, not APY-style compounding rates (see §2.1). The
naming is retained for continuity with contract source code and the
established user-facing convention. The mitigation is the UI requirement
to display absolute reward amounts alongside any rate display, so that a
user can verify the exact tokens they will receive without reasoning
about compounding semantics.

### 16.6 Migration Reserve Crowding-Out

`migrationReservedRewards` reduces `availableRewards` during the
migration window. This is necessary to guarantee that every pre-TGE
participant can materialise their position on-chain regardless of the
order in which they claim. The trade-off is that early post-TGE stakers
see a smaller available pool than the gross reward inventory would
suggest. The effect is bounded by the size of pre-TGE commitments. Materialisation
does not by itself free reward capacity: the obligation moves from
`migrationReservedRewards` to `reservedRewards`. Capacity improves through
new reward refills, or, for unclaimed pre-TGE obligations, through the
migration-reserve release process after the migration window, three-month
grace period, and 14-day notice have elapsed.

### 16.7 Pool-Health Floor User Visibility

When `rewardInventory` — not `availableRewards` — falls below 30% of
`REFERENCE_POOL`, the SmartVest pool-health floor activates and can
raise sell-side transfer fees above the normal curve when the floor
exceeds the otherwise-applicable fee (see SmartVest specification for
the formula). This is the self-correcting refill mechanism in operation
when fee-bearing transfer volume and vault distribution conditions exist:
higher effective fees increase vault accumulation, distributions refill
the pool, and restored pool health lowers the floor pressure.
The visibility trade-off is that users may interpret elevated fees
negatively even though their economic purpose is to support the staking
system that protects holder commitments. Mitigation is on the UI side: a
visible pool-health indicator that explains the temporary elevation.

---

## 17. Status

This specification corresponds to the Phase 1 deployment planned for
Token Generation Event on Ethereum L1. The contract is currently in
Phase 2 integration testing alongside the other nine FROGEN contracts.
Subject to refinement through audit cycles and tier-1 third-party
security audit. Final parameter values will be confirmed at deployment.

---

## Disclaimer

This specification is published for informational and technical-reference
purposes only. It describes a smart contract mechanism and does not
constitute an offer to sell, a solicitation to buy, or a recommendation
regarding any token, security, or financial instrument. Nothing in this
document is investment, financial, legal, tax, or accounting advice.

The FROGEN token and the DynamicStaking contract are experimental
software. Interacting with them carries technical risks including but
not limited to smart contract bugs, economic-mechanism failures,
governance actions, key compromise, network congestion, regulatory
changes, and total loss of funds. No representation or warranty,
express or implied, is made as to the accuracy, completeness, or
fitness for any purpose of this specification or the underlying
contracts. The specification may differ from the deployed
implementation; the deployed bytecode is authoritative for actual
behaviour.

Participation in FROGEN may be restricted in certain jurisdictions.
Readers are responsible for determining whether their use of the
protocol complies with the laws applicable to them and should consult
qualified professional advisors before acting on any information in
this document.
GenLabs Ltd. and its affiliates accept no liability for any loss or
damage arising from reliance on this specification or from any use of
the FROGEN protocol.
