# Activity Shaping

Armada constrains transaction sizes so most activity flows through common payment ranges, where legitimate demand is highest. This builds dense anonymity sets where users need them, while leaving large transfers statistically visible as outliers. We call this "activity shaping."

Armada's protocol stores governance-set activity shaping parameters on-chain (such as transaction size constraints and recommended ranges), which the SDK and integrators can read as coordination defaults. Enforcement of specific policies (like limits per transaction or per route) is implemented by relayers, integrators, and reference applications. The base protocol does not embed identity screening or discretionary transaction censorship.

## Design Philosophy

Privacy infrastructure faces a fundamental tradeoff: the same properties that protect everyday users can also obscure unwanted activities. Rather than attempting to screen users or judge intent, Armada takes a mechanism-design approach, shaping transaction flows so that the anonymity set is strongest where real usage is densest.

This means:

- **High-volume, everyday payment ranges** benefit from large anonymity sets and strong privacy guarantees
- **Large, exceptional transfers** face thinner anonymity sets and higher statistical visibility
- Misuse at scale becomes economically conspicuous without requiring identity checks or permission systems

The result is a system optimized to maximize legitimate utility, rather than subsidizing large, low-frequency extraction events.

## How It Works

Armada's ecosystem applies flow constraints that concentrate shielded activity within specific transaction ranges. These constraints are not about policing users. They shape which anonymity sets grow and which remain sparse.

When most activity flows through a narrow band, transactions within that band blend into a large crowd. Transactions outside it stand out statistically, even if they remain technically private.

| Transaction Type | Anonymity Set | Privacy Strength |
|------------------|---------------|------------------|
| Common payment ranges | Dense | Strong |
| Uncommon / large transfers | Sparse | Weak |

This is not a guarantee against misuse. It is a structural bias that makes Armada useful for payments and poor for concealment at scale.

## Why This Matters

The social good of private payments--protecting individuals from surveillance, enabling financial autonomy, reducing data breach exposure--must far outweigh the residual risk of small-scale misuse that cannot be structurally prevented.

Traditional approaches to preventing misuse in privacy systems tend toward one of three failure modes:

1. **Open systems** that provide equal privacy regardless of transaction characteristics, enabling large-scale concealment
2. **Permissioned systems** that require identity verification, defeating the purpose of privacy infrastructure
3. **Hybrid systems** that provide privacy as a two-tiered system based on exclusion criteria that are controlled by a trusted party

**Activity shaping** preserves privacy for the flows that matter most to users, while ensuring that attempts to hide large thefts stand out (by producing statistically anomalous patterns).

## Mechanism Toolkit

Activity shaping is not a single feature. It is a set of mechanism-design levers, implemented across protocol defaults, integrator policies, and reference application behavior. The base protocol accepts arbitrary amounts; most activity shaping is applied at the layers above it.

### Transaction size constraints

Relayers and integrators enforce maximum transfer sizes that concentrate shielded activity in common payment ranges. Transactions within those ranges blend into dense anonymity sets. Transactions outside them (where available via alternative routes) face thinner sets and weaker privacy.

Default parameters for reference infrastructure are set through governance. Integrators can customize constraints for their specific use cases.

### Rate limits

Per-address or per-time-window rate limits constrain how quickly value can flow through the system, making large-scale extraction operationally slow. Fragmentation attacks (splitting a large transfer into many small ones at the policy limit) produce detectable patterns — timing bursts, correlated outflows, and repeated transactions at ceiling amounts.

### Ingress normalization

Integrators and reference applications can channel public deposits into a small set of standard transfer amounts, reducing the amount-uniqueness signal available to observers.

When pools are young and dwell times are short, deposit amount is one of the strongest correlation signals an adversary can use to link deposits to withdrawals. Channeling ingress into common amounts removes that signal class. This is most valuable during early network formation, when timing cover is thin and organic flow density has not yet developed.

Ingress normalization is enforced at the integrator/application layer, not the protocol. The protocol accepts arbitrary amounts. See [Ingress Normalization](INGRESS_NORMALIZATION.md) for implementation details and phasing.

### Fee design

Shield fees create an economic gradient that shapes which flows enter the system. Volume-tiered pricing rewards integrators who drive routine activity, reinforcing density in high-utility ranges. See [Fee Structure](FEE_STRUCTURE.md).

### Withdrawal behavior

Arbitrary withdrawals are supported, but withdrawal patterns (amount, timing, recurrence) are a leakage surface. Relayer policies, optional batching windows, and note-selection logic in reference wallets can reduce behavioral fingerprinting on the withdrawal side.

### Integrator defaults

Integrators set their own policy constraints. The ecosystem converges on effective shaping through integrator competition and governance-set defaults, not protocol-level rigidity.

## Frequently Asked Questions

**Could Circle freeze USDC in Armada?**
Armada's USDC deposits carry issuer risk, because Circle can blacklist contract addresses. We've [analyzed precedent](USER_CONSIDERATIONS.md) and believe this risk is low for Armada's design. Activity Shaping makes Armada a poor tool for large-scale laundering, the use case that triggers sanctions. Circle has only blacklisted under government order, not proactively.

**Does Armada block large transactions?**

The base protocol does not include identity-based blocking. However, relayers/integrators may enforce policy constraints (eg. max per transfer) to keep shielded activity concentrated in high-utility ranges. Users can still transact outside typical ranges via alternative routes where available, but those flows will generally have smaller anonymity sets.

**Can someone circumvent activity shaping by splitting transactions?**

Fragmentation attacks tend to produce detectable patterns (highly repeated withdrawals at the policy limit, timing bursts, and correlated outflows). This does not guarantee attribution, but it makes large-scale exit behavior more statistically conspicuous than in systems designed to provide uniform privacy at all sizes.

**How is this different from compliance screening?**

Compliance screening attempts to identify and exclude bad actors. Activity shaping does not screen anyone. It structures incentives so that the system naturally serves its intended use case better than alternative uses.

**Doesn't channeling deposits into fixed amounts look like a mixer?**

Fixed-denomination deposits were Tornado Cash's entire privacy mechanism. In Armada, ingress normalization is one lever among many, applied at the integrator layer on top of a system that supports arbitrary shielded balances, transfers, and withdrawals. The protocol has no denomination concept. Integrators channel public deposit amounts to reduce correlation signals, not to define privacy "notes." The distinction is structural: Tornado's denominations *are* the anonymity set. Armada's anonymity set is the entire shielded pool; ingress normalization just reduces one class of observable signal at the entry point.

**What about regulatory requirements?**

Activity Shaping is a product- and network-design strategy, not a compliance system. It's designed so that Armada's privacy guarantees are concentrated where they provide the most social value.