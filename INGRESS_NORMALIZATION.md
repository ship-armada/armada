# Ingress Normalization

Ingress normalization channels public deposits into a small set of standard transfer amounts to reduce amount-based correlation. It is one lever in Armada's [Activity Shaping](ACTIVITY_SHAPING.md) toolkit, implemented at the integrator/application layer.

This document specifies ingress normalization for the Armada front-end (the reference application). Other integrators may adopt, adapt, or ignore these policies.

## Problem

When a user deposits an idiosyncratic amount (eg. 2,347 USDC) and later a withdrawal of a similar amount appears, the amount itself is a correlation signal. The rarer the deposit amount, the stronger the signal.

This problem is worst when:

- Pool activity is thin (few deposits to hide among)
- Dwell times are short (deposit-to-withdrawal timing window is narrow)
- Timing cover is weak (few concurrent transactions to blend with)

All three conditions hold during early network formation. Amount uniqueness is one of the few leakage channels that can be suppressed before the system has organic volume.

## Non-goals

This mechanism does not:

- Change the core protocol's support for arbitrary amounts
- Guarantee anonymity on its own
- Replace other activity-shaping levers (fee design, withdrawal constraints, relayer policy, integrator defaults)
- Attempt to optimize for large irregular flows

## Design Principles

1. **Protocol neutrality.** The base protocol continues to support arbitrary amounts. Ingress normalization is not a protocol concern.
2. **Public ingress coordination.** The Armada front-end shapes the publicly visible deposit amount, since that is where amount uniqueness is exposed. Shielded balances remain arbitrary.
3. **Small number of bands.** Fewer bands means denser sets. Too many bands fragments the crowd and defeats the purpose.
4. **Single public ingress event.** The Armada front-end prefers one public deposit at one standard amount. Public multi-deposit synthesis is discouraged.
5. **Bootstrap-first logic.** Early on, stronger coordination matters more than optionality.
6. **Relax over time.** As crowd density improves, hard banding can evolve into softer recommendations.

## Bootstrap Policy

### Core rule

During Bootstrap, the Armada front-end accepts deposits only at standard ingress amounts and prefers a single public deposit event. Public multi-deposit synthesis is not the default path and should be avoided because it creates a secondary observable pattern.

The user selects a standard public deposit amount. Approximating arbitrary targets through multiple public deposits is discouraged during Bootstrap.

**Deposit amounts are not denominations.** Once funds enter the shielded pool, they exist as arbitrary shielded balances. There are no "notes" at standard sizes. The normalization applies only to the publicly visible deposit event. If internal note decomposition is useful for spend management, it happens inside the shielded domain after ingress.

### Bootstrap ingress amounts

⚠️ Subject to change. Exact amounts should be validated against expected early use patterns and integrator profiles.

| Amount | Target use case |
|--------|-----------------|
| 100 USDC | Small payments, testing |
| 1,000 USDC | Routine payments, payroll components |
| TBD | Larger payments, treasury operations |

The third amount is the biggest unresolved design choice. Candidates:

| Option | Argument for | Argument against |
|--------|-------------|-----------------|
| 5,000 USDC | Lower ceiling forces more concentration into 1,000 band; matches the Armada front-end's payment focus | May be too low for treasury operations; pushes larger depositors to other integrators or multiple deposits |
| 10,000 USDC | Covers treasury use cases; cleaner logarithmic spacing | Wider spread may thin out the middle range; large deposits may remain sparse regardless |

A fourth mid-range amount (eg. 2,500) has been considered and deferred. Adding it cuts expected density per band by ~25% at the same volume. It is easier to add a band later than to remove one.

All deposit types — payment shields and shield-to-yield — flow through the same public ingress surface and contribute to the same band density. Yield-oriented deposits strengthen cover for payment users and vice versa.

### Why no arbitrary public deposits in Bootstrap

Allowing arbitrary public deposit amounts during bootstrap weakens one of the few controllable anti-linkage levers available early. When the pool is young, you do not have the luxury of hoping users organically create good crowd structure. You either shape the crowd early or accept weak early privacy.

### Why no public multi-deposit synthesis in Bootstrap

If a user wants 2,300 USDC, the Armada front-end should not default to publicly sending 2×1,000 + 3×100. Multi-deposit sequences create a secondary observable pattern — the combination itself can fingerprint the depositor, especially when:

- The combination is unusual
- The same user repeats the same combination
- The deposits arrive in rapid succession

A user wanting an amount not covered by a single standard amount should deposit at the nearest standard amount and accept the difference. In Growth phase, custom deposit amounts become available.

## Phasing

| Phase | Behavior | Trigger |
|-------|----------|---------|
| Bootstrap | Hard normalization — the Armada front-end only accepts deposits at standard amounts; single deposit per ingress event | Launch through early growth |
| Growth | Semi-hard — standard amounts presented as defaults; custom available behind advanced settings with clear privacy warning | Governance decision based on pool density metrics |
| Mature | Adaptive — SDK recommends amounts with strongest current cover; custom always available | Sufficient organic flow density |

Phase transitions are governance decisions, not automatic. The criteria should be based on observed anonymity set density within standard bands, not calendar time.

### Fallback: multi-deposit in later phases

Once the pool has sufficient density (Growth phase or later), multi-deposit paths may be offered as a non-default option. If implemented, they should:

- Introduce timing jitter between deposits (randomized delay within a configurable window)
- Avoid deterministic ordering (don't always deposit largest-first)
- Warn users that multi-deposit patterns are weaker than single deposits

This is future-phase guidance, not Bootstrap policy.

## Withdrawal Side

Ingress normalization does not constrain withdrawals. Withdrawals remain arbitrary.

This is a deliberate asymmetry, not an oversight. The reasoning:

**Usability.** Constrained withdrawals would significantly degrade the core use case. Payments require arbitrary amounts. If a user needs to pay 2,137 USDC, they need to withdraw 2,137 USDC.

**Withdrawal amount is not the primary signal once ingress is normalized.** The relevant candidate set for a withdrawal is broader than deposits at a matching public amount — once ingress is normalized, an adversary cannot match a withdrawal to a specific deposit by amount alone. Timing correlation within a band remains a threat, addressed through other shaping levers.

**Timing and behavioral patterns are the stronger withdrawal-side leakage vectors.** These are better addressed through other shaping levers: relayer policies, optional batching windows, wallet-side note selection that minimizes rare recomposition patterns, and avoiding deterministic "withdraw immediately after deposit" behavior.

Critics may argue that normalizing deposits while leaving withdrawals unconstrained "leaves the back door open." The honest answer is that withdrawal leakage exists and should be addressed — but through levers appropriate to the withdrawal side, not by crippling payment usability.

## UX Guidance

### Do

- Present amounts as "deposit amounts" or "transfer amounts," not "denominations" or "note sizes"
- Frame the constraint as "these amounts have the strongest privacy right now"
- Explain that standard amounts exist to strengthen early crowd cover — users should understand why
- Show qualitative cover levels ("strong cover" / "moderate cover") rather than precise anonymity set counts
- In Growth phase, show custom amounts with a clear, non-patronizing explanation of reduced cover

### Don't

- Use denomination / note / bucket language in the UI
- Present this as a permanent constraint — it's a bootstrap measure
- Show precise anonymity set counts (these are gameable by adversaries and misleading to users)
- Hide the reason for the constraint

### Copy direction

Instead of: "Choose a denomination"

Something like: "Select a deposit amount. Standard amounts blend with other deposits for stronger privacy."

Instead of: "Custom public deposit amounts are disabled"

Something like: "The Armada front-end currently uses common deposit sizes to strengthen early crowd cover. Custom amounts will be available in a future phase."

## Governance Interface

Governance controls for reference infrastructure:

| Parameter | Description |
|-----------|-------------|
| Standard ingress amounts | The set of accepted public deposit amounts |
| Phase | Current ingress normalization phase (Bootstrap / Growth / Mature) |
| Custom deposit availability | Whether custom amounts are available and under what conditions |

These parameters apply to governance-controlled reference infrastructure (the Armada front-end, default relayer configuration). Integrators set their own policies independently.

## Metrics and Monitoring

The Armada front-end should track:

- Deposit count per ingress band
- Share of total deposits by band
- Median dwell time by band
- Fraction of users withdrawing within a short window after deposit
- Concentration ratio (percentage of deposits in top 1–2 bands)
- Frequency of low-traffic bands
- Evidence of distinct user behavioral repetition (consistent band + time pattern combinations)

**Success looks like:**

- Dense concentration in a few routine ingress amounts
- Low amount uniqueness in public deposit data
- Growing dwell times as users gain confidence

**Failure looks like:**

- Too many bands with weak usage
- User drop-off from excessive rigidity
- Visible repeated patterns that remain easy to correlate despite normalization

## Risks

**Crowd fragmentation.** Too many ingress bands weakens the whole mechanism. Start with fewer than feels natural.

**UX friction.** Some users will dislike not being able to deposit exact amounts. The explanation needs to be honest and the constraint needs to visibly relax as the pool matures.

**Mixer optics.** Fixed deposit amounts can evoke Tornado Cash comparisons. Language and framing matter — see [Activity Shaping FAQ](ACTIVITY_SHAPING.md#frequently-asked-questions) for the structural distinction.

**False confidence.** Normalized ingress helps, but weak timing cover can still dominate. Ingress normalization is one lever, not a privacy guarantee.

**Repeated behavior.** If users always choose the same band on predictable schedules, behavioral linkage may still be strong. Wallet-side guidance should discourage recognizable patterns.

## Open Questions

1. **Bootstrap band set.** Three amounts or four? Which ones? The third amount (5,000 vs 10,000) is the biggest unresolved choice, dependent on expected early integrator profiles and use cases.
2. **Adversary information surface.** Can pool-relative context (eg. "1,000 USDC deposits are common right now") be shown to users without giving adversaries a free timing/amount correlation tool? May need to restrict to qualitative indicators.
3. **Phase transition criteria.** What anonymity set density within standard bands is "enough" to relax into Growth phase? This needs a concrete metric, not a vibes check.
4. **Integrator divergence.** Should specific integrators be encouraged to maintain stricter ingress normalization even after the Armada front-end relaxes? Divergent policies across integrators could either strengthen the system (different integrators serve different risk profiles) or fragment it (users cluster by integrator policy, creating sub-pools).
5. **Yield dwell-time profiles.** Yield-oriented deposits contribute to band density (stated above), but their dwell-time profiles may differ significantly from payment users. If yield deposits dominate a band and have long dwell times while payment deposits are short-lived, the effective anonymity set for short-dwell payment users may be thinner than the raw band count suggests. Worth monitoring but may not require different treatment.
