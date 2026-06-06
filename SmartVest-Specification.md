# SmartVest Specification

**Version:** 1.0
**Date:** 2026-06-07
**Authors:** FROGEN Team @ GenLabs Ltd.
**Status:** Canonical specification, subject to refinement through audit cycles.
**Repository:** https://github.com/frogen-protocol/frogen-specs
**License:** See repository [LICENSE](./LICENSE).

---

## 1. Overview

SmartVest is the fee-calculation subsystem of the FROGEN protocol. It determines
the algorithmic core component of the unified transfer fee that the FROGEN token
applies to fee-bearing transfers. SmartVest holds no tokens and moves no tokens.
It is a deterministic calculator: given the current lock percentage of the supply
and the current health of the staking reward pool, it returns the core fee rate,
in basis points, that the token should apply. The token combines this core rate
with a separately governed ecosystem base rate to form the total transfer fee, and
routes the resulting fee tokens to the SmartVest vault. The accumulation and
distribution of those tokens are the responsibility of the vault subsystem and are
specified separately.

The core fee is adaptive. It starts at a governance-set ceiling when little of the
supply is locked and decays linearly as the locked fraction rises, reaching a small
permanent minimum that keeps the active core-fee rate from decaying to zero; actual
refill flow still depends on fee-bearing transfer volume, successful vault
distribution, and transfer-size rounding. A
second, independent mechanism can raise the core fee above its normal level when the staking reward inventory is critically low, if the pool-health minimum exceeds the normal curve value, scaling the increase to the size of the
deficit so that fee-driven refills accelerate exactly when the pool most needs them.
Except where the token-side core-fee pause short-circuits the core fee (§8.1), both
behaviours are evaluated on every fee-bearing transfer from current on-chain state. Governance does not, and cannot, set the live rate directly; its only levers
are two bounded parameters and the curve shape itself is fixed.

SmartVest is immutable in architecture and logic, with a small bounded governance
surface for two numeric parameters. It is not deployed behind a proxy and exposes no
upgrade path; the decay curve, the permanent minimum, the pool-health response, the
rounding rules, the constants, and the parameter bounds are fixed at deployment and
cannot be changed by any account or governance action. Against that fixed structure,
two parameters remain adjustable within hard-coded limits through a timelocked role:
the core-fee ceiling and the ecosystem base fee. This is a deliberate design pattern,
in the same family as adjustable risk parameters on otherwise-fixed lending or
exchange protocols: the rules are permanent, and only a pair of bounded dials can move.
Within the wider FROGEN architecture SmartVest is read by the
[FROGEN token](Cross-Contract-Specification.md#frogen-token) on each fee-bearing transfer, takes
its lock and pool-health inputs from the
[staking subsystem](Cross-Contract-Specification.md#dynamicstaking),
the [SmartVest vault](Cross-Contract-Specification.md#smartvestvault), and the
[circulating-supply tracker](Cross-Contract-Specification.md#circulatingsupplytracker),
and is governed by the protocol TimelockController.

---

## 2. Terminology and Notation

### 2.1 Fee Terminology

> **Terminology note.** Three distinct quantities are referred to as "fees" in
> this specification and must not be conflated. The **SmartVest core fee**
> (`coreFeeBps`) is the algorithmic, decaying component that SmartVest computes.
> The **ecosystem base fee** (`baseFeeBps`) is a separate, fixed component governed
> independently and added by the token. The **unified transfer fee**
> (`totalFeeBps`) is their sum, a rate in basis points; the token deducts the corresponding token
> amounts by computing `coreFeeAmount` and `baseFeeAmount` independently, as specified
> in §10. SmartVest is responsible only for the core fee. It reports
> the base fee as a stored value for convenience, but the base fee is not affected
> by the decay curve, the permanent minimum, or the pool-health response.

### 2.2 Glossary

| Term | Definition |
|---|---|
| `coreFeeBps` | The SmartVest core fee rate for a transfer, in basis points. The output of SmartVest's calculation. |
| `baseFeeBps` | The ecosystem base fee rate, in basis points. A fixed, governance-bounded value stored on SmartVest and added by the token. |
| `totalFeeBps` | The unified transfer fee rate: `coreFeeBps + baseFeeBps`. Computed and applied by the token. |
| `maxCoreFeeBps` | The governance-set ceiling on the core fee, in basis points. Scales the entire decay curve. The live core fee never exceeds this value. |
| `lockPercentageBps` | The fraction of circulating supply currently locked in the staking and vault subsystems, in basis points. Computed by the token and passed to SmartVest. |
| `totalLocked` | `vaultBalance + totalStakedPrincipal`. The token amount treated as locked for the purpose of the fee curve. Staking reward inventory is excluded. |
| `circulatingSupply` | Total token supply minus provably transfer-restricted balances, as defined by the circulating-supply tracker. The denominator of `lockPercentageBps`. |
| `poolHealthBps` | `floor(rewardInventory × 10000 / REFERENCE_POOL)`. The staking reward inventory as a fraction of the reference pool, in basis points. Computed by the token and passed to SmartVest. |
| `rewardInventory` | The staking contract's token balance minus its total staked principal. The pool of reward tokens. The input to `poolHealthBps`. |
| `CAP_FLOOR_BPS` | The permanent curve-wide minimum core fee. 25 bps. Immutable. |
| `REFERENCE_POOL` | The fixed denominator of the pool-health metric. 155,400,000 × 10^18 tokens. Immutable, and shared by value with the token and staking subsystems. |

### 2.3 Notation

- Basis-point (`Bps`) values are integers where 10000 represents 100.00%.
- Token amounts use the FROGEN token's 18-decimal precision.
- All division is integer floor division. The curve and floor never use any
  rounding mode other than floor.
- The two inputs to the core-fee calculation, `lockPercentageBps` and
  `poolHealthBps`, are computed by the token from on-chain state and supplied to
  SmartVest. SmartVest does not query other contracts on the transfer path. Its
  view-only convenience accessor derives both quantities itself for off-chain use
  only; see §5.3.

---

## 3. Core Mechanism

### 3.1 What SmartVest Computes

SmartVest answers one question on the transfer path: given how much of the supply
is locked and how healthy the staking reward pool is, what core fee rate applies?
The answer is a deterministic mapping from two inputs and one governance-bounded
parameter. It involves no stored-state writes, no token movement, and no external
calls. Because the calculation is deterministic and side-effect-free, the same inputs
always produce the same
output, and the result can be reproduced and audited independently from on-chain
state.

The token, not SmartVest, owns the surrounding behaviour: it computes the two
inputs, applies conservative defaults if any state read fails (§9.1), adds the base
fee, computes the actual token amounts (§10), and moves the fee tokens to the vault.
This separation is deliberate. The token controls the failure behaviour of its own
state reads, and SmartVest remains a small, immutable, independently verifiable
calculator.

### 3.2 The Two Fee Components

The unified transfer fee has two components with different governance and different
dynamics.

- The **core fee** is algorithmic. It decays as the locked fraction rises and can be
  pushed back up by the pool-health response. It is the subject of this
  specification.
- The **base fee** is fixed. It is a single governance-bounded value, unaffected by
  lock percentage or pool health. SmartVest stores it and reports it; the token adds
  it to the core fee.

Keeping the two separate means the protocol can fund ecosystem obligations through a
predictable flat component while the core component does the adaptive supply and
refill work.

### 3.3 Constants

| Constant | Value | Role |
|---|---|---|
| `CAP_FLOOR_BPS` | 25 bps | Permanent minimum core fee. Applied as a curve-wide floor via a maximum against the decayed value, so the decay curve never produces less than 25 bps regardless of lock percentage. Not governable. |
| Pool-health threshold | 3000 bps (30%) | The `poolHealthBps` value at or above which the pool-health response is dormant. Below it, the response can raise the core fee. Not governable. |
| Lock reference point | 5000 bps (50%) | The lock percentage at which the linear decay term reaches zero. Above it the decayed term is zero and the permanent minimum governs. Not governable. |
| `REFERENCE_POOL` | 155,400,000 × 10^18 tokens | Denominator of the pool-health metric. Must equal the reference pool value used by the token and the staking subsystem. Immutable. |

`REFERENCE_POOL` is a denominator, not a cap on actual reward inventory. The staking
pool may hold more than this through fee refills or other injections; the excess does
not change the core-fee calculation beyond driving `poolHealthBps` to or above 100%,
where the pool-health response is dormant.

---

## 4. Mathematical Specification

### 4.1 Linear Decay (Raw Core Fee)

The decayed core fee is linear in the locked fraction up to the lock reference point,
and zero at or above it:

```
rawCoreFeeBps = floor(maxCoreFeeBps × (5000 − lockPercentageBps) / 5000)   if lockPercentageBps < 5000
rawCoreFeeBps = 0                                                          if lockPercentageBps >= 5000
```

Equivalently, below the reference point the raw core fee is `maxCoreFeeBps` scaled by
the unlocked fraction `(5000 − lockPercentageBps) / 5000`. At 0% lock it equals
`maxCoreFeeBps`; at 50% lock the linear term reaches zero.

### 4.2 Curve-Wide Minimum (Normal Core Fee)

A permanent minimum is applied across the entire curve:

```
normalCoreFeeBps = max(CAP_FLOOR_BPS, rawCoreFeeBps)
```

Because the minimum is applied as a maximum against the decayed value rather than as
a separate branch at the reference point, the curve has no discontinuity. The linear
term decays smoothly until it falls to the 25 bps minimum, after which the minimum
governs. This crossover occurs at approximately 48.1% lock for an illustrative `maxCoreFeeBps` of 650 bps. For other ceilings, the crossover is the first integer `lockPercentageBps` at which `floor(maxCoreFeeBps × (5000 − lockPercentageBps) / 5000) < 25`; lower ceilings reach the floor earlier, and higher ceilings reach it later. From the crossover through
100% lock the normal core fee holds flat at 25 bps.

### 4.3 Pool-Health Response

Independently of lock percentage, a minimum core fee is derived from the staking
reward pool's health and applied as a second floor. It is dormant when the pool is at
or above the 30% threshold and scales with the deficit below it:

```
minCoreFeeBps = 0                                                       if poolHealthBps >= 3000
minCoreFeeBps = floor(maxCoreFeeBps × (3000 − poolHealthBps) / 3000)    if poolHealthBps < 3000
```

At exactly 30% health the response contributes nothing; at 0% health it contributes
`maxCoreFeeBps`, the maximum possible core fee.

### 4.4 Effective Core Fee and Unified Fee

The core fee returned by SmartVest is the larger of the normal core fee and the
pool-health minimum:

```
coreFeeBps = max(normalCoreFeeBps, minCoreFeeBps)
```

The token then forms the unified transfer fee by adding the base fee:

```
totalFeeBps = coreFeeBps + baseFeeBps
```

The two floors compose as a two-layer safety net. The curve-wide minimum guarantees
at least 25 bps of core fee at any lock percentage. The pool-health response can lift
the core fee well above that when the reward pool is depleted, and relaxes back to the
curve as refills restore the pool. Because both are applied as a maximum, the effective
core fee is always at least the normal core fee and never exceeds `maxCoreFeeBps`.

### 4.5 Lock Percentage and Pool Health Inputs

The two inputs are computed by the token from on-chain state. The lock percentage uses
the locked principal and vault balance over circulating supply, excluding staking
reward inventory:

```
lockPercentageBps = floor(totalLocked × 10000 / circulatingSupply),   totalLocked = vaultBalance + totalStakedPrincipal
```

If `circulatingSupply` is zero or unavailable, the token sets `lockPercentageBps` to
zero, which yields the maximum decayed core fee rather than a bypass (§9.1). Pool
health uses the staking reward inventory over the reference pool:

```
poolHealthBps = floor(rewardInventory × 10000 / REFERENCE_POOL),   rewardInventory = balanceStaking − totalStakedPrincipal
```

If the staking principal read fails, the token cannot distinguish principal from
rewards and sets `poolHealthBps` to zero, which activates the pool-health response at
full strength (§9.1).

### 4.6 Rounding and Integer Math

Every division in this specification is integer floor division. The decay term, the
pool-health term, and the input ratios all round down. The permanent minimum and the
two-layer maximum are applied after rounding. The raw decay and pool-health fee divisions never round their own outputs upward.
Because `lockPercentageBps` and `poolHealthBps` are themselves floored before use, the
final integer fee can differ by up to one basis point from a hypothetical calculation
on the exact unfloored ratios; the floored integer bps inputs are canonical.

### 4.7 Properties

Three properties hold for all inputs within the governance bounds and were confirmed
by exhaustive evaluation over the input grid.

- **Bounded by the ceiling.** `coreFeeBps <= maxCoreFeeBps` for every
  combination of lock percentage and pool health. The pool-health response can reach
  `maxCoreFeeBps` at zero health but never exceeds it.
- **Monotone in lock.** With the pool healthy (at or above 30%), `coreFeeBps` is
  non-increasing in `lockPercentageBps`: locking more supply never raises the core
  fee.
- **Permanent refill floor.** `coreFeeBps` is at least 25 bps at every lock
  percentage, so the core fee rate never falls to zero while the fee is active. Actual refill revenue still depends on fee-bearing transfer volume, distribution execution, and transfer-size rounding.

### 4.8 Worked Examples

The following two tables are illustrative and assume `maxCoreFeeBps = 650 bps`. The
deployed ceiling is set at deployment within the governable range (§11); substituting a different ceiling scales the raw linear term and the pool-health minimum proportionally, while effective values clamped by `CAP_FLOOR_BPS` remain 25 bps. Lock and pool-health
figures shown are exact integer outputs of the formulas above.

**Decay curve at a healthy pool (pool-health response dormant).**

| Lock % | Decayed term (bps) | Effective core fee (bps) |
|---|---|---|
| 0% | 650 | 650 |
| 2% | 624 | 624 |
| 5% | 585 | 585 |
| 8% | 546 | 546 |
| 10% | 520 | 520 |
| 15% | 455 | 455 |
| 20% | 390 | 390 |
| 25% | 325 | 325 |
| 30% | 260 | 260 |
| 40% | 130 | 130 |
| 48% | 26 | 26 |
| 48.10% | 24 | 25 (minimum governs) |
| 50%+ | 0 | 25 (minimum governs) |

**Pool-health response at a fixed 45% lock (normal core fee 65 bps).**

| Pool health % | Pool-health minimum (bps) | Effective core fee (bps) |
|---|---|---|
| 30%+ | 0 (dormant) | 65 |
| 25% | 108 | 108 |
| 20% | 216 | 216 |
| 15% | 325 | 325 |
| 10% | 433 | 433 |
| 5% | 541 | 541 |
| 0% | 650 | 650 |

In the first table the pool is healthy, so the effective core fee equals the decay
curve, transitioning smoothly to the 25 bps minimum at approximately 48.1% lock. In the second the
lock is high enough that the decay curve alone would produce 65 bps, but a depleted
pool lifts the effective fee through the pool-health response, reaching the ceiling at
zero health. As distributions refill the pool above 30% health, the response goes
dormant and the fee returns to the curve value.

---

## 5. Fee Determination

This section describes the core-fee calculation as a mechanism. It is a read-only
computation with no state change.

### 5.1 Inputs

The token supplies two values: the current `lockPercentageBps` and the current
`poolHealthBps`, each computed as in §4.5. The governance-bounded ceiling
`maxCoreFeeBps` is read from SmartVest's own state.

### 5.2 Sequence

1. Compute the raw decayed core fee from the ceiling and the lock percentage (§4.1).
   At or above the lock reference point the decayed term is zero.
2. Apply the permanent minimum to obtain the normal core fee (§4.2).
3. Compute the pool-health minimum from the ceiling and the pool health (§4.3). At or
   above the 30% threshold this is zero.
4. Return the larger of the normal core fee and the pool-health minimum as the
   effective core fee (§4.4).

The token adds the base fee to this result to form the unified fee, then computes
token amounts and routes them; those steps are token behaviour, with fee-amount computation summarised in §10 and transfer-path failure behaviour in §9.1.

### 5.3 Read-Only Current-Rate Accessor

SmartVest exposes a view-only accessor that reports the current core fee rate for
interfaces and dashboards. Unlike the transfer-path calculation, this accessor derives
`lockPercentageBps` and `poolHealthBps` itself from current on-chain state and returns
the core fee. The unified rate a user pays is obtained by adding the separately readable base fee, which SmartVest also exposes. It is intended for off-chain display only and is not
used by the token on the transfer path, because the token must control the failure
defaults on its own state reads rather than depend on SmartVest's reads. Interfaces
that display the rate should treat this accessor as a convenience and reconcile it
against the token's applied fee where exactness matters.

---

## 6. Governance Parameter Updates

SmartVest holds two mutable parameters. Each is updated through a timelocked role and
validated against fixed bounds; an out-of-bounds update reverts. Beyond these two parameters, the only mutable state is role membership, administered
as described in §9; no account can change the curve shape, the constants, the bounds,
or the live rate directly.

### 6.1 Maximum Core Fee

The core-fee ceiling `maxCoreFeeBps` may be set within the governable range [100, 850]
bps through the timelocked role. It scales the entire decay curve and the pool-health
response proportionally. It does not affect the permanent 25 bps minimum, which is a
constant, nor the base fee.

### 6.2 Base Fee

The ecosystem base fee `baseFeeBps` may be set within the governable range [0, 100]
bps through the timelocked role. It is independent of the core-fee curve and of
`maxCoreFeeBps`. Setting it to zero removes the base component from the unified fee;
the core component continues to apply.

### 6.3 What Governance Cannot Do

The following are fixed at deployment and cannot be changed by any role, any
combination of roles, or any on-chain action:

- The linear decay formula, the permanent minimum, the pool-health response formula,
  the rounding rules, and the lock reference point.
- The 25 bps permanent minimum, the 30% pool-health threshold, and `REFERENCE_POOL`.
- The governable bounds themselves: the [100, 850] and [0, 100] ranges cannot be
  widened.
- The live core fee rate, which is always the algorithmic output and is never set
  directly.

---

## 7. Edge Cases

| Case | Behaviour |
|---|---|
| Lock at or above 50% | The decayed term is zero; the permanent 25 bps minimum governs the normal core fee. The pool-health response can still raise the effective fee if the pool is below 30% health. |
| Pool health at or above 30% | The pool-health response is dormant and contributes nothing; the effective core fee equals the decay curve with its 25 bps minimum. |
| Pool health zero (read failure or empty pool) | The pool-health minimum equals `maxCoreFeeBps`; the effective core fee is the ceiling. This is the conservative result of the token's fail-closed handling when staking principal cannot be read (§9.1). |
| Circulating supply zero or unavailable | The token sets `lockPercentageBps` to zero, so the decayed term is at its maximum and the full core fee applies. There is no bypass via an induced supply read failure. |
| Calculation reverts on the transfer path | The token applies `maxCoreFeeBps` as its failure default for the core fee, preserving transfer liveness without allowing a bypass (§9.1). |
| Base fee read fails on the transfer path | The token applies a zero base fee as its failure default, prioritising transfer liveness; the core fee is unaffected (§9.1). |
| Very small transfer | The token computes core and base amounts independently by floor division; either component can round to zero tokens for sufficiently small transfers. SmartVest returns rates, not amounts; the rounding occurs in the token's amount computation (§9.1, §10). |
| Governance value at a bound | An update exactly at 100, 850, or the base-fee bounds is accepted; an update outside the range reverts. |

---

## 8. Integration Points

This section describes how SmartVest interacts behaviourally with other FROGEN
contracts. Implementation-level call surfaces are out of scope; the corresponding
specifications and the
[Cross-Contract Specification](Cross-Contract-Specification.md) govern those.

### 8.1 FROGEN Token

The token is the only consumer of SmartVest on the transfer path. On each fee-bearing
transfer the token computes `lockPercentageBps` and `poolHealthBps` from on-chain
state, reads SmartVest for the core fee using those two inputs, reads SmartVest for
the base fee, forms the unified fee, computes the token amounts, and routes them to
the vault. The token, not SmartVest, owns all failure defaults on these reads: if the
core-fee calculation reverts it applies `maxCoreFeeBps`; if the base-fee read reverts
it applies zero; if the supply or principal reads fail it applies the conservative
lock and pool-health defaults described in §9.1. The intent throughout is that no induced read failure in the core-fee input or calculation path can lower the core fee below the conservative default. A base-fee read failure is the explicit liveness exception: it can temporarily waive only the base component. See the
[Cross-Contract Specification](Cross-Contract-Specification.md#frogen-token).

Two token-side behaviours interact with the core fee without being part of SmartVest's
calculation. First, the protocol's emergency core-fee pause is a token-side mechanism:
when it is engaged and the reward pool is healthy (pool health at or above the 30%
threshold), the token short-circuits and applies a zero core fee without calling
SmartVest at all. When the pool is below the threshold, the token calls SmartVest
regardless, so the pool-health response can still raise fees during distress even while
the pause is engaged; SmartVest itself has no awareness of the pause flag and always
returns its normal output. Second, the base fee continues to apply during a core-fee
pause unless governance has separately set it to zero. Neither behaviour changes
SmartVest's logic; both are described here because they determine the core fee a user
actually pays.

### 8.2 SmartVest Vault

SmartVest computes rates; the vault holds and distributes the resulting tokens.
SmartVest does not read the vault on the transfer path. The vault's balance does enter
the fee curve indirectly, because the token includes the vault balance in
`totalLocked` when it computes `lockPercentageBps`. Distribution of accumulated fees,
including the split between the staking reward pool and other recipients, is governed
by the vault and specified separately. See the
[SmartVest Vault Specification](Cross-Contract-Specification.md#smartvestvault).

### 8.3 DynamicStaking

The staking subsystem provides the two pieces of state behind SmartVest's inputs: the
total staked principal, used both in `totalLocked` for the lock percentage and as the
subtrahend for reward inventory, and, through the staking contract's token balance, the
reward inventory used for pool health. Staking reward inventory is excluded from
`totalLocked` so that protocol-held rewards do not register as user lock pressure. The
read-only current-rate accessor (§5.3) holds a reference to the staking contract to
derive pool health for off-chain display. See the
[DynamicStaking Specification](Cross-Contract-Specification.md#dynamicstaking).

### 8.4 Circulating-Supply Tracker

The denominator of the lock percentage, `circulatingSupply`, is supplied by the
circulating-supply tracker, which excludes provably transfer-restricted balances. The
token reads the tracker when computing `lockPercentageBps` and applies the zero-supply
default if the tracker is unavailable (§7, §9.1). See the
[Circulating-Supply Tracker Specification](Cross-Contract-Specification.md#circulatingsupplytracker).

**VestingManager and PrizeTimeLock.** SmartVest and the token do not read vesting or
prize-timelock contracts on the transfer path. Their balances affect the core fee only
through the circulating-supply denominator: balances held in contracts that provably
restrict transfer (including team and treasury vesting and the prize timelock) are
treated as non-circulating and excluded from `circulatingSupply` while restricted, with
their classification and timing defined by the circulating-supply tracker. Such
balances are never part of `totalLocked`, which is `vaultBalance + totalStakedPrincipal`;
vested or timelocked tokens enter `totalLocked` only if they are subsequently staked as
principal or held by the SmartVest vault.

### 8.5 TimelockController

The protocol TimelockController is the governance authority over SmartVest's two
parameters and over role administration. See §9.

### 8.6 Deployment and Wiring

Because SmartVest is immutable, it is deployed directly rather than behind a proxy. Its contract-reference addresses are fixed at construction and have no setters; a reference recorded incorrectly cannot be changed in place and would require the redeploy-and-migrate remediation of §9.5. Its two initial fee parameters are constructor inputs that may afterward be adjusted only within the hard bounds described in §6 and §11.2.

Two deployment-time invariants concerning SmartVest must hold before any transfers are processed: SmartVest's `REFERENCE_POOL` must equal the value used by the token and the staking subsystem (§11.1), and the staking-contract address SmartVest records must equal the staking-contract address the token records.
After deployment, privileged roles on SmartVest are held by the TimelockController and
the deployer holds no residual role (§9.2). The full deployment procedure, ordering,
constructor argument lists, and post-deployment verification are governed by the
[Cross-Contract Specification](Cross-Contract-Specification.md); this section states
only what is needed to understand how SmartVest is situated.

---

## 9. Governance and Access Control

### 9.1 Token-Side Failure Behaviour

Although failure handling is the token's responsibility, it is integral to SmartVest's
security posture and is recorded here for completeness. On the transfer path the token
wraps its state reads so that no single failure reverts a transfer, and every core-fee-path default is conservative; the base-fee read default is a liveness exception:

- Staking principal read fails: `poolHealthBps` is set to zero, activating the
  pool-health response at full strength. The default is set directly rather than
  derived, because a defaulted principal of zero against a valid staking balance would
  overstate reward inventory and understate the deficit.
- Circulating-supply read fails: `lockPercentageBps` is set to zero, applying the
  maximum decayed core fee.
- Core-fee calculation reverts: the token applies `maxCoreFeeBps`.
- Base-fee read reverts: the token applies a zero base fee, prioritising transfer
  liveness over base-fee collection.

These defaults converge on the protocol-protective outcome. A read failure can cause
legitimately reduced fees to be temporarily higher, but cannot be used to lower or bypass the core fee. A base-fee read failure can temporarily waive the base component, as documented above.

### 9.2 Role Overview

SmartVest uses role-based access control with a single operational role.

- **Timelocked role.** Authorised to update `maxCoreFeeBps` and `baseFeeBps` within
  their bounds. At launch this role is held by the TimelockController, so updates are
  delayed by the controller's configured delay, 48 hours at launch. If governance
  later grants this role directly to another account, that account could invoke the
  updates without a per-call controller delay; such a grant should be treated as
  delegating the corresponding authority after the grant proposal's own timelock delay.
- **Role administration.** The default admin role administers all SmartVest role membership: it can grant and
  revoke the timelocked role, and it can grant, revoke, or renounce the default admin
  role itself. It is held by the TimelockController, so these role changes are
  themselves subject to the timelock delay. As with the timelocked role, granting the
  default admin role directly to another account would delegate role-administration
  authority after that grant's own timelock, after which that account's role changes
  would not carry a per-call controller delay. There is no emergency role on SmartVest and no pause:
  the emergency core-fee pause that exists in the protocol is a token-side mechanism,
  not a SmartVest function.

At deployment the privileged roles are assigned to the TimelockController, and no
deployer or externally owned account retains a privileged role on SmartVest. This
matches the protocol-wide deployment posture in which the deployer holds no residual
authority after deployment.

### 9.3 Capabilities

The complete set of governance-callable operations on SmartVest is:

- Update `maxCoreFeeBps` within [100, 850] bps (timelocked role; reverts out of
  bounds).
- Update `baseFeeBps` within [0, 100] bps (timelocked role; reverts out of bounds).
- Administer role membership through the default admin role: grant or revoke the
  timelocked role, and grant, revoke, or renounce the default admin role itself
  (default admin held by the TimelockController; timelocked, with the direct-grant
  caveat in §9.2).

### 9.4 Limitations

The following are absent from SmartVest and cannot be exercised by any role, any
combination of roles, or any on-chain action:

- Change the decay curve, the permanent minimum, the pool-health response, the
  rounding rules, the lock reference point, the 30% threshold, or `REFERENCE_POOL`.
- Widen the governable bounds on either parameter.
- Set the live core fee rate directly, bypass the curve, or disable the permanent
  minimum.
- Pause SmartVest or upgrade its logic. SmartVest has no pause and no upgrade path.

### 9.5 Immutability and Remediation Path

SmartVest is not deployed behind a proxy. Its code cannot be replaced, and it exposes
no upgrade entry point. There is consequently no proxy-upgrade trust assumption on
SmartVest itself: the decay curve, the minimum, the pool-health response, and the
bounds are permanent, and the only post-deployment changes possible are the two
bounded parameter updates.

The same immutability sets the remediation path. Because the token holds its reference
to SmartVest as a fixed value and the token is itself immutable, a defect in SmartVest's
logic cannot be patched in place and cannot be repointed to a corrected SmartVest by
governance. Remediation requires deploying a new SmartVest and a new token that
references it, which is a token migration. This is the highest-bar remediation in the
system and is discussed as a trade-off in §12.1.

---

## 10. Token-Side Fee Application

SmartVest returns rates. The conversion of rates into token amounts is performed by
the token and is summarised here because it determines what a user actually pays and
is the origin of the small-transfer rounding behaviour referenced in §7.

The token computes the two fee amounts independently by floor division of the transfer
amount against each rate, then deducts their sum:

```
coreFeeAmount = floor(transferAmount × coreFeeBps / 10000),   baseFeeAmount = floor(transferAmount × baseFeeBps / 10000)
```

```
recipientReceives = transferAmount − coreFeeAmount − baseFeeAmount
```

Computing the two amounts independently keeps each component's rounding deterministic
and ensures the accounting identity
`transferAmount = recipientReceives + coreFeeAmount + baseFeeAmount`
holds exactly. Because each amount floors independently, either component can round to
zero tokens for a transfer small enough that the product is below 10000, which is the
mechanical reason a sufficiently small transfer can carry no fee for one or both
components. The fee tokens are routed to the vault with the core and base portions
reported separately so the vault can account for them independently.

---

## 11. Parameter Reference

SmartVest's values fall into three categories: values fixed permanently in the
contract, values governance may adjust within hard-coded bounds, and the initial
launch values supplied at deployment. The categories are listed separately to make the
governance surface unambiguous.

### 11.1 Permanently Immutable

These are fixed at deployment and cannot be changed by any account or governance
action. The decay formula, the curve-wide-minimum rule, the pool-health response
formula, and the rounding rules are likewise permanent and have no numeric entry.

| Value | Setting | Role |
|---|---|---|
| `CAP_FLOOR_BPS` | 25 bps | Permanent curve-wide minimum core fee. |
| Pool-health threshold | 3000 bps (30%) | `poolHealthBps` at or above which the pool-health response is dormant. |
| Lock reference point | 5000 bps (50%) | Lock percentage at which the linear decay term reaches zero. |
| `REFERENCE_POOL` | 155,400,000 × 10^18 tokens | Pool-health denominator. Must equal the value used by the token and the staking subsystem. |
| `maxCoreFeeBps` bounds | [100, 850] bps | The governable range for the core-fee ceiling. The bounds themselves cannot be widened. |
| `baseFeeBps` bounds | [0, 100] bps | The governable range for the base fee. The bounds themselves cannot be widened. |
| Constructor reference addresses | Set once at construction | SmartVest's references to the other protocol contracts. No setter exists for any of them; the exact set is enumerated in the Cross-Contract Specification. |

### 11.2 Governance-Adjustable Within Hard Bounds

These two parameters may be changed through the timelocked role, each constrained to
its immutable bounds. An out-of-bounds update reverts.

| Parameter | Bounds | Change path | Effect |
|---|---|---|---|
| `maxCoreFeeBps` | 100–850 bps | Timelocked role; 48-hour delay at launch (§9.2) | Scales the entire decay curve and the pool-health response proportionally. |
| `baseFeeBps` | 0–100 bps | Timelocked role; 48-hour delay at launch (§9.2) | The fixed ecosystem base fee, added by the token. Independent of the core-fee curve. |

### 11.3 Initial Launch Values

These are supplied to the constructor at deployment, within the bounds in §11.2, and
become the starting values that governance may later adjust.

| Parameter | Launch value | Status |
|---|---|---|
| `maxCoreFeeBps` | 650 bps (6.50%), within [100, 850] | Launch value; to be confirmed at deployment. |
| `baseFeeBps` | 50 bps (0.50%), within [0, 100] | Launch value; to be confirmed at deployment. |

The launch value of `maxCoreFeeBps` is a deployment-configured constructor input, not
a fixed constant. Substituting any value within the governable range scales the raw linear term and pool-health minimum proportionally, but does not scale values already clamped by the fixed 25 bps floor. The floor crossover moves according to the formula in §4.2; the shape of the curve and the properties in §4.7 are unchanged.

---

## 12. Design Trade-offs

### 12.1 Immutability: Stronger on Upgrade Risk, Weaker on Remediation

SmartVest's immutability is a deliberate trade-off with two sides, and both are
disclosed here.

On the strength side, SmartVest carries no proxy-upgrade trust assumption of its own.
The decay curve, the permanent minimum, the pool-health response, and the parameter
bounds cannot be changed by governance or any other party after deployment. The only
post-deployment levers are the two bounded parameter updates, each constrained to a
fixed range that cannot itself be widened. A holder reasoning about how the core fee
will behave can rely on the curve shape being permanent, which is a stronger guarantee
than an upgradeable contract can offer.

On the weakness side, immutability raises the cost of fixing a defect to its maximum.
Because both SmartVest and the token that references it are immutable, a logic defect
in SmartVest cannot be patched in place and cannot be repointed to a corrected
contract. The only remediation is to deploy a new SmartVest together with a new token
referencing it and migrate, which is a token migration and the highest-bar remediation
in the system. The mitigation is concentration of audit effort: an immutable
fee-calculation core should receive the highest audit priority precisely because it
cannot be corrected after launch.

Between those two poles sits the bounded governance surface, which is neither an
upgrade path nor a frozen constant. Governance can adjust exactly two numeric
parameters, each within hard-coded bounds that cannot be widened, through a timelocked
role that at launch is held by the TimelockController and so carries its 48-hour
public delay; a later direct grant of that role is itself timelocked, but subsequent
calls by the grantee would not carry a per-call delay (§9.2). It cannot change the formulas, the bounds
themselves, the constants, the constructor references, or anything else. Role
administration is itself bounded the same way: the default admin role that grants and
revokes the timelocked role is held by the TimelockController, so a change to who may
adjust the parameters is also subject to the 48-hour delay, and the deployer renounces
any privileged role after deployment as a deployment invariant. This is the
"immutable architecture and logic, with a small bounded governance surface" pattern
stated in §1: the rules are permanent, two dials can move within fixed limits and, while the timelocked role is held by the
TimelockController, only after public notice (§9.2), and there is no mechanism to do
more.

### 12.2 Conservative Fail-Closed Defaults

Every core-fee failure default on the transfer path resolves toward the maximum core fee. The base-fee read is the documented exception and defaults to zero to preserve transfer liveness
(§9.1). The benefit is that no induced state-read failure can be used to bypass or
reduce the fee. The cost is that during a genuine dependency outage, legitimately
reduced fees can be temporarily higher than honest state would produce, and the
pool-health response can activate at full strength if the staking principal read fails.
This is accepted because transfer liveness is preserved throughout and the
elevated-fee state ends once the dependency recovers. Fees collected during the outage
are not automatically refunded or reversed; they remain subject to the normal vault
accounting and distribution flow.

### 12.3 Small-Transfer Rounding

Because the token computes each fee component by independent floor division, a
sufficiently small transfer can carry no fee for one or both components (§10). In
principle a large transfer could be split into many micro-transfers to reduce total
fees paid. On a network where per-transfer gas cost exceeds the fee saved, this is
uneconomical; on a very low-cost network the trade-off shifts. The behaviour is
accepted rather than mitigated, because a minimum-fee floor would penalise legitimate
small transfers and add complexity disproportionate to the benefit.

### 12.4 Pool-Health Response Visibility

When the staking reward inventory falls below 30% of `REFERENCE_POOL`, the pool-health response can raise the core fee above the curve value when its computed floor exceeds the otherwise-applicable fee (§4.3). This is the self-correcting refill mechanism operating under fee-bearing transfer volume and vault distribution conditions: higher effective fees increase vault accumulation, distributions refill the reward pool, restored pool health relaxes the response, and the fee returns to the curve. The trade-off is that users may read elevated fees negatively even though their
purpose is to sustain the staking system. The mitigation is on the interface side: a
visible pool-health indicator that explains the temporary elevation.

### 12.5 Governance Ceiling Within Bounds

Governance can move `maxCoreFeeBps` within [100, 850] bps through the timelocked role,
which scales the whole curve and the pool-health response. This is a real lever over
the fee that holders experience, and it is disclosed as such. It is bounded on both
sides by limits that cannot be widened, it cannot change the curve shape or the
permanent minimum, and it is subject to the timelock delay, so any change routed through the TimelockController is bounded, public, and
observable in advance; a direct grant of the role carries the trust implications
described in §9.2. The same public notice window also means scheduled fee changes are
economically anticipatable: users and bots may accelerate transfers before a scheduled
increase, delay them before a scheduled decrease, or target the execution boundary.
The timelock provides transparency and review time, not protection against timing
behaviour around the change.

### 12.6 Dependence on Upgradeable Inputs

SmartVest itself is immutable, but the inputs it consumes originate in contracts that
are upgradeable behind proxies, including the vault, the staking subsystem, and the
supply tracker. SmartVest's own outputs are a fixed function of its inputs, so a claim
about how SmartVest transforms inputs into a core fee is absolute. A claim about the
end-to-end fee a user pays is correct under the canonical implementation and short of a
proxy upgrade of those upgradeable dependencies, whose own specifications and upgrade
trust assumptions govern the values SmartVest reads.

---

## 13. Status

This specification corresponds to the Phase 1 deployment planned for the Token
Generation Event on Ethereum L1. The contract is in integration testing alongside the
other FROGEN contracts. It is subject to refinement through audit cycles and tier-1
third-party security audit. Final parameter values, including the launch value of
`maxCoreFeeBps`, will be confirmed at deployment.

---

## Disclaimer

This specification is published for informational and technical-reference
purposes only. It describes a smart contract mechanism and does not
constitute an offer to sell, a solicitation to buy, or a recommendation
regarding any token, security, or financial instrument. Nothing in this
document is investment, financial, legal, tax, or accounting advice.

The FROGEN token and the SmartVest contract are experimental
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
