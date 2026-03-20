# How the team earns governance power

When the crowdfund closes, only one group governs Armada: the people who funded it.

Team and airdrop tokens exist on-chain from day one. They are published, visible, and accounted for. But they carry zero voting power and cannot be transferred until the protocol generates real revenue. No calendar. No vesting cliff. No unlock without earnings.

This is the arrangement: you fund it and you govern it, and decide who is responsible for allocating the funds and if that should change. You decide if you still believe in the mission, or if you want to wind down the protocol and distribute remaining treasury to ARM holders.

---

## The shift

As cumulative protocol revenue grows, team and airdrop tokens unlock in tranches and begin voting. Here is what that looks like:

| Cumulative Protocol Revenue | Crowdfund participants | Launch team (individuals) | Airdrop cohort |
|---|---|---|---|
| $0 — launch | 100% | 0% | 0% |
| $10k | 83% | 13% | 4% |
| $50k | 67% | 25% | 8% |
| $100k | 56% | 33% | 11% |
| $250k | 46% | 41% | 14% |
| $500k | 38% | 46% | 15% |
| $1M — full unlock | 33% | 50% | 17% |

_Assumes 10% crowdfund distribution (base raise). If the crowdfund expands to 15% (1.8M ARM), crowdfund participants hold 43% at full unlock — equal to the launch team._

A few things the table doesn't show that matter:

**Launch team tokens are held in individual wallets, not a shared entity.** Allocations are distributed to individual team members before the crowdfund opens and are published. Each person governs independently. The 50% column represents the aggregate if every team member voted identically on every proposal — which is both unlikely and visible on-chain when it happens.

**The airdrop cohort has its own independent interests and views.** The 17% at full unlock is held by recipients whose prior work informed the protocol — each governing independently, not as a bloc aligned with the launch team.

**No individual team member holds outsized unilateral power.** The largest single allocation in the launch team is 4.5% of total supply. At full unlock that represents 15% of non-treasury votes — meaningful, but not controlling on its own.

At full unlock, the launch team in aggregate holds a governance majority only by building a protocol that generates $1M in cumulative protocol fees. By earning it, not by waiting for a scheduled unlock.

---

## What counts as revenue

Protocol revenue accrues from two sources:

**Shield fees.** Charged at deposit — 50bps by default, adjusted by governance. Every dollar shielded generates fee income. $1M in shield fees alone represents roughly $200M in total deposit volume over the protocol's life.

**Yield fees.** Armada takes 15% of yield earned on capital held in the shielded pool by default, adjusted by governance. Long-term depositors contribute to revenue passively. A sustained $22M pool at 3% APY generates ~$100k per year in yield fees.

Revenue from any fee mechanism approved by governance counts toward the same cumulative total. The milestone counter doesn't distinguish between sources — it measures trajectory. Is the protocol generating real income from real usage, and is that income growing over time?

What the thresholds represent in practice:

| Milestone | What it likely signals |
|---|---|
| $10k | Protocol is live. Real deposits. Not a demo. |
| $100k | A handful of integrators actively routing volume. |
| $1M | Sustained institutional usage. The model is working. |

---

## The failure case

If revenue milestones are never reached, locked tokens never unlock. Team and airdrop allocations have no claim on treasury assets in a wind-down. Only circulating ARM holders — the people who paid — have priority.

There is no floor for zero-cost-basis tokens. There is no time-based fallback. The team's tokens are worth something only if the protocol is worth something.

---

## The manipulation question

A reasonable concern: could someone deposit a large sum specifically to trigger token unlocks cheaply, then withdraw?

At 50bps, accumulating $100k in revenue requires $20M in deposit volume — real capital at risk, not a rounding error. Withdrawing a large deposit also requires many small transactions by design. The protocol concentrates privacy guarantees around everyday payment sizes; extracting a large position through small withdrawals is expensive in time and gas. Capital that enters the pool is structurally sticky unless the depositor is a genuine long-term participant. In which case, they're contributing exactly the kind of usage that justifies the unlock.

---

## The terms, plainly stated

You are funding a pre-product protocol at a fixed price. You govern it alone at launch. The team has economic upside only if the protocol earns. Your governance position dilutes only as they prove it's worth diluting.

If this works, everyone earns. If it doesn't, those who took the real financial risk have priority on what remains.

That's the deal. It's written into the contracts before the crowdfund opens.
