# Armada Crowdfunding Spec

## Overview

Armada will raise funds by "word-of-mouth whitelisting." The allocation mechanism is algorithmic and predeclared. Once the commitment window opens, participant allocations are determined entirely by the rules of the contract, not by launch team discretion. The initial network shape (who is invited as a seed, and who the launch team invites directly) is determined by the launch team before and during the window. This is intentional and disclosed.

### Parameters

| Parameter | Value |
|---|---|
| Purpose | Fund MVP |
| Minimum fund | $1,000,000 USDC |
| Base fund | 1,200,000 ARM ($1.2M) |
| Max fund (elastic) | 1,800,000 ARM ($1.8M) |
| Expansion trigger | $1,500,000 in capped demand |
| Percent of supply | 10 - 15% |
| Price | $1.00 per ARM |
| Timing | Pre-product |

---

## Token Distribution

**Total supply: 12,000,000 ARM**

| Allocation | ARM | % | Unlock | Voting Power |
|---|---|---|---|---|
| Crowdfund | 1,200,000 - 1,800,000 | 10-15% | Governance proposal | Active once claimed and delegated |
| Team | 1,800,000 | 15% | Revenue milestones | Revenue-gated |
| Airdrop | 600,000 | 5% | Revenue milestones | Revenue-gated |
| Treasury | 7,800,000 - 8,400,000 | 65-70% | Governance-controlled | None |

### Pre-Crowdfund Distribution

Before the crowdfund opens, the following revenue-locked allocations are distributed and published:

**Team Allocation (1,800,000 ARM):** See Team Allocation section below for full specification. All assignments are revenue-locked and published — role labels, wallet addresses, and percentages — before the crowdfund opens.

**Airdrop Allocation (600,000 ARM):**
- Distributed to qualifying recipients before the crowdfund opens
- Revenue-locked (same terms as team)

**Transparency:**

Crowdfund participants can evaluate the full distribution (ie. who holds what and under what terms) before committing.

### Team Allocation

The team allocation (1,800,000 ARM, 15%) is held in a single shared revenue-lock contract with per-beneficiary allocations. Each team member has their own entry in the beneficiary list and governs independently — the team does not vote as a bloc.

The Knowable Safe appears in the beneficiary list like any other team member, with a larger allocation reserved for future contributors. Future contributor distributions are handled off-chain (token agreements between Knowable and contributors); once ARM is released to the Safe's wallet per the milestone schedule and global transfers are enabled, Knowable distributes via standard transfers. See REVENUE_LOCK.md for the full contract spec.

**Properties:**
- Revenue-locked transferability (same schedule as all team/airdrop tokens)
- Voting power unlocks proportionally with revenue milestones
- Each team member governs independently ie. the team does not vote as a bloc

**Constraints:**
- Cannot change the revenue-lock schedule
- Cannot pull from treasury or other allocations
- Fluid pool distributions inherit the same unlock terms

### Revenue-Gated Unlocks (Team + Airdrop)

Team and airdrop tokens unlock and gain voting power only via cumulative protocol revenue:

| Cumulative Revenue | Unlocked | Voting Power |
|---|---|---|
| $0 | 0% | 0% |
| $10k | 10% | 10% |
| $50k | 25% | 25% |
| $100k | 40% | 40% |
| $250k | 60% | 60% |
| $500k | 80% | 80% |
| $1M | 100% | 100% |

Governance power follows economic commitment. Until the protocol generates revenue, only people who paid decide.

---

## Word-of-Mouth Whitelist

### Design Philosophy

This is a trusted-network crowdfund, not viral growth. Participation requires invitation from someone closer to the core. Hop-0 and hop-1 each have an enforced ceiling — the maximum fraction of the available pool each can absorb. Hop-2 has no enforced ceiling; instead it has a reserved floor (first claim on 5% of the fund) and an effective ceiling determined by the floor plus any hop-1 leftover. Actual allocation is demand-driven: each hop receives whatever it committed, up to its effective ceiling, and anything unused rolls forward to later hops. The launch team's invite budget closes after week 1; seeds and hop-1 invitees may invite throughout all three weeks; commitments remain open for the full three weeks.

### Initial Network Seeding

The people creating Armada are inviting their network of allies to help form the Armada community. Seeds, and the initial invitations from the launch team, are the first nodes in a trust network that extends outward through word of mouth.

Seed criteria are determined by the launch team based on demonstrated alignment, contribution, and long-term interest in privacy infrastructure.

**Maximum seed count: 150.** The total number of seeds is capped at 150. This bounds the maximum network size at approximately 1,740 participant slots (150 seed nodes + up to 510 hop-1 nodes + up to 1,080 hop-2 nodes). Arithmetic: hop-1 slots = (150 seeds × 3 invites) + 60 launch-team hop-1 = 510; hop-2 slots = (510 hop-1 nodes × 2 invites) + 60 launch-team hop-2 = 1,080. Because the same address may occupy multiple hop levels, the number of distinct individuals may be lower.

**Seed additions.** The launch team may add new seeds at any point during the first week, up to the 150-seed cap. Seeds added later in week 1 naturally have less of the 3-week commitment window remaining — that's the cost of late addition. Once added, a seed may issue invitations throughout the full three weeks.

**Launch team hop-1 invite budget.** In addition to seeds, the launch team holds a predeclared budget of 60 direct hop-1 placements, not tied to any seed slot. This allows the launch team to invite allies with direct relationships to hop-1 without creating phantom unfilled hop-0 slots. The budget is fixed for the duration of the crowdfund.

Launch team hop-1 invitees are full hop-1 participants: same cap ($4,000), same invite rights (up to 2 hop-2 invites each). They appear in the invite graph with the launch team address as inviter. In the `(address, hop)` node model, launch-team-issued invites originate from a designated `(launch_team_address, ROOT)` sentinel node — a fixed address with no hop level and no cap, whose sole role is to issue the predeclared launch team budget. This node is not a participant and makes no commitment.

**Launch team hop-2 invite budget.** The launch team also holds a predeclared budget of 60 direct hop-2 placements. The intended use case is friends, family, and adjacent supporters for whom the $1,000 cap is appropriate. These participants receive no invite rights (consistent with all hop-2 participants). Budget is fixed for the duration of the crowdfund.

**No financial incentive to invite.** Participants receive no allocation benefit from others' commitments. There is no referral bonus, no token reward, and no increase in a participant's own allocation from someone else's commitment. Invitations are an act of community-building, not a financial strategy. This keeps the invite graph honest: people invite those they genuinely want in the community, not those most likely to maximize their own return.

### Hop Structure

| Hop | Description | Ceiling | Floor | Cap | Invites |
|---|---|---|---|---|---|
| 0 | Seeds (max 150) | 70% | — | $15,000 | 3 |
| 1 | Seed invites + launch team invitations | 45% | — | $4,000 | 2 |
| 2 | Hop-1 invites + launch team invitations | — ¹ | 5% | $1,000 | 0 |

¹ Hop-2 has no enforced ceiling — only a floor. Its effective ceiling is `hop2_floor + hop1_leftover`, which can reach the full sale size if hop-0 and hop-1 are both empty. The 10% figure shown in some earlier drafts was removed as dead code (`HOP_CEILING_BPS` contains no entry for hop-2).

The launch team holds a predeclared invite budget separate from hop-0: 60 hop-1 invitations and 60 hop-2 invitations. These do not consume seed invite slots. Launch team invitations may only be issued during the first week.

Hop-2 has first claim on 5% of the fund before hop-0 and hop-1 ceilings are applied. This reserved capacity is always available to hop-2, regardless of earlier hop demand, but hop-2 only receives it to the extent hop-2 participants actually commit. Hop-0 and hop-1 ceilings are calculated against the fund net of this reservation. At base fund: $60k reserved for hop-2, hop-0 and hop-1 compete for the remaining $1.14M.

Hop-0 and hop-1 ceilings are overlapping by design — they sum to 115% of the available pool, not 100%. Hop-2 has no enforced ceiling; its effective ceiling is its reserved floor plus any hop-1 leftover, which can range from the floor to the full sale size. No hop is guaranteed to fill its ceiling. Each hop absorbs up to its effective ceiling based on actual demand; anything unused rolls forward. Earlier hops fill first; later hops absorb whatever remains.

### Ceiling Amounts

Hop-2 floor reservation comes off the top. Hop-0 and hop-1 ceilings are applied against the remainder.

**Base fund ($1.2M):**

| Hop | Floor reservation | Available pool | Ceiling (enforced) | Max ceiling (with rollover) |
|---|---|---|---|---|
| 2 | $60,000 | — | No enforced ceiling | $60k floor + full hop-1 leftover |
| 0 | — | $1,140,000 | 70% = $798k | $798k |
| 1 | — | $1,140,000 | 45% = $513k | $1,140k (if hop-0 empty) |

**Expanded fund ($1.8M):**

| Hop | Floor reservation | Available pool | Ceiling (enforced) | Max ceiling (with rollover) |
|---|---|---|---|---|
| 2 | $90,000 | — | No enforced ceiling | $90k floor + full hop-1 leftover |
| 0 | — | $1,710,000 | 70% = $1,197k | $1,197k |
| 1 | — | $1,710,000 | 45% = $769.5k | $1,710k (if hop-0 empty) |

### Invitation Limits

Invitation limits are hop-specific and predeclared:

- Hop-0 → may invite up to 3 addresses to hop-1
- Hop-1 → may invite up to 2 addresses to hop-2
- Hop-2 → may not invite

Invitations are signed on-chain (inviter → invitee, with hop level). The invite graph is defined over `(address, hop)` nodes, not bare addresses. This means:

- The same address may appear at multiple hop levels as a distinct node at each
- An invite edge connects `(inviter_address, inviter_hop)` → `(invitee_address, invitee_hop)`
- A seed (hop-0) inviting their own address to hop-1 creates a self-loop edge, visible as such in the graph
- Invite slot consumption is per `(address, hop)` node: a hop-0 node has 3 outgoing invite slots; a hop-1 node has 2. A single address occupying both hop-0 and hop-1 has both slot budgets, consumed independently. **There is no limit on how many times a given address may be invited to the same hop** — each invite creates a separate participation slot, and the address's effective cap at that hop equals `participation_slots[(address, hop)] × HOP_CAP[hop]`. Invite rights also scale: each invite to hop-1 grants 2 outgoing hop-2 invite slots. A natural consequence: a seed who self-invites their own address to hop-1 three times occupies 3 hop-1 slots (cap: 3 × $4k = $12k) and gains 6 outgoing hop-2 invite slots — giving one address unilateral control over a full $33k subtree, fully legible in the graph. For full entity-level capture analysis, see Self-Filling section.
- For allocation, each `(address, hop)` node participates independently — an address at hop-0 and hop-1 receives ARM from both hop buckets, accumulated into a single address balance

This model makes entity-level participation unambiguously legible without wallet correlation. Separate-wallet self-fill is also permitted and visible in graph shape, but same-entity control across wallets is not provable on-chain.

**An address may appear in multiple independent subtrees.** There is no constraint preventing address_a from being a seed in their own right (hop-0) while also being invited by an unrelated seed to hop-1 or hop-2. Each `(address, hop)` node participates in allocation independently, so address_a accumulates ARM from every position it holds across all subtrees. All such participation is visible in the graph — every edge is public — so participants can observe multi-subtree presence directly. This is a deliberate choice: we cannot prevent it, and allowing the same address makes concentration legible rather than hidden.

### Invite Mechanism

Invitations can be issued through two paths. Both produce the same on-chain result — an `Invited` event and a new edge in the invite graph — but they differ in who initiates the transaction and when the invitee's address is known.

#### Path A: Invite Link (signature-based, primary)

The inviter creates a shareable link without knowing the recipient's address. The invitee opens the link, connects their wallet, and joins the network by committing USDC — all in a single atomic transaction.

**How it works:**

1. The inviter signs an EIP-712 typed data message authorizing one invite slot. No gas, no on-chain transaction. The message contains:
   - `inviter`: the inviter's address
   - `fromHop`: the inviter's hop level (determines the invitee's hop as `fromHop + 1`)
   - `nonce`: per-inviter unique identifier (any `uint256 > 0`; nonce 0 is reserved for direct invites; each nonce can be consumed or revoked once; UI generates sequential nonces starting from 1)
   - `deadline`: block timestamp after which the signature is invalid (default: 5 days from creation)

2. The signed message is encoded into a URL and shared privately (DM, email, Signal, etc.).

3. The invitee opens the link, connects their wallet, enters a commitment amount, and submits a single transaction: `commitWithInvite(inviter, fromHop, nonce, deadline, signature, amount)`.

4. The contract verifies the signature, checks the nonce and deadline, records the invite edge, records the commitment, and emits both `Invited` and `Committed` events atomically.

**Properties:**
- **Bearer credential.** The link can be used by anyone who has it. The inviter does not choose the recipient — the first person to redeem the link becomes the invitee. Share links privately.
- **No slot reservation.** Signing a link does not reserve an invite slot. Slots are checked at redemption time. An inviter may sign more links than they have slots; only the first N redeemed (where N = available slots) will succeed.
- **Invite and commitment are atomic.** The invitee appears in the graph only when they commit with funds. There are no invited-but-uncommitted phantom nodes from the link path.
- **5-day default expiry.** The `deadline` is baked into the signed message. Expired links fail at redemption. Generating a replacement link is free (another off-chain signature).
- **Revocable.** The inviter may call `revokeInviteNonce(nonce)` to permanently invalidate a link before it is redeemed. This is an on-chain transaction (costs gas) and serves as a kill-switch for leaked links.

**Signature verification:** The contract uses OpenZeppelin's `SignatureChecker.isValidSignatureNow()`, which supports both EOA wallets (via `ecrecover`) and smart contract wallets (via EIP-1271 `isValidSignature`). This ensures multisig and smart wallet users can create invite links.

**EIP-712 domain (all four fields validated by the contract):**

```
domain: {
  name: "ArmadaCrowdfund",
  version: "1",
  chainId: <deployment chain>,
  verifyingContract: <crowdfund contract address>
}
```

All four domain fields must be validated to prevent cross-chain and cross-contract signature replay.

#### Path B: Direct Invite (on-chain, fallback)

The inviter enters the invitee's address and calls `invite(invitee, fromHop)` directly, where `fromHop` is the inviter's hop level and the invitee joins at `fromHop + 1`. The invite is recorded on-chain immediately. The invitee appears in the graph with $0 committed and can commit separately at any time before the deadline.

**Properties:**
- **Address-targeted.** The inviter must know the invitee's address at invite time.
- **Inviter pays gas.** The invitee pays nothing until they choose to commit.
- **Graph presence before commitment.** Unlike the link path, the invitee appears in the graph immediately — even if they never commit. This is accepted because the inviter is making a deliberate, addressed placement.

Both paths produce the same graph edge — `(inviter, fromHop) → (invitee, fromHop + 1)` — and the same `Invited` event. The difference is operational: links are for "share a link with someone you want in the community"; direct invites are for "I know this person's address and want to place them now."

#### Duplicate Invites

Multiple inviters may invite the same address to the same hop. Each invite creates a new participation slot for that address at that hop. The invitee's effective cap at the hop increases by `HOP_CAP[hop]` per invite received — this is the same `participation_slots[(address, hop)] × HOP_CAP[hop]` rule that applies to self-invites. All edges are recorded and visible in the graph. An invite slot is consumed for each inviter regardless.

The same inviter may also invite the same address to the same hop more than once (e.g., via two separate links). Each successful redemption creates a new slot. This is permitted for consistency ("allow everything, make it visible"), and the inviter's slot budget limits how many such invites they can issue.

#### Invite Revocation

Invites issued via the direct path (`invite()`) are irrevocable — once the on-chain transaction confirms, the invitee has a permanent position.

Invite links (Path A) can be revoked before redemption by calling `revokeInviteNonce(nonce)`. This permanently invalidates the nonce. The revoked link will fail if anyone attempts to redeem it. Revocation does not free the invite slot — the slot is not consumed until a link is successfully redeemed.

### Self-Filling and Max Single-Entity Capture

A participant may commit at multiple hop levels using a single address, and receives ARM allocation from each hop they participate in — subject to that hop's per-slot cap and pro-rata rules. This is the preferred model for transparency: a seed using the same address across all hops creates unambiguously legible self-loop edges in the invite graph, and receives their full multi-hop allocation without any penalty.

There is no per-address limit on how many times an address may appear at a given hop. Each invite to a hop creates a separate participation slot, and the effective cap equals `participation_slots[(address, hop)] × HOP_CAP[hop]`. A seed using all 3 of its hop-1 invite slots on itself gains 3 hop-1 slots (cap: 3 × $4k = $12k), and each of those slots grants 2 hop-2 invites — 6 hop-2 slots total (cap: 6 × $1k = $6k).

Maximum self-fill for a seed via full recursive self-invitation with one address:

| Hop | Per-slot cap | Slots (self-invite) | Max USDC (single address) |
|-----|-------------|---------------------|--------------------------|
| Hop-0 | $15,000 | 1 | $15,000 |
| Hop-1 | $4,000 | 3 (from hop-0 invites) | $12,000 |
| Hop-2 | $1,000 | 6 (from hop-1 invites) | $6,000 |
| **Total** | | | **$33,000** |

An address may also receive invites from other participants (e.g., another seed inviting the same address to hop-1), which adds further slots beyond the self-invite tree. The per-hop cap for any address is `participation_slots[(address, hop)] × HOP_CAP[hop]` — there is no per-address ceiling beyond what the invite graph produces. An address invited by multiple independent seeds accumulates across all subtrees; all such participation is visible in the graph.

Participating across hops using separate wallets is also permitted. Graph edges are visible, but same-entity control across separate wallets cannot be proven on-chain.

A seed committing at all three hops via full recursive self-invitation gets ARM allocation at all three — up to $15k from hop-0, $12k from hop-1 (3 slots × $4k), and $6k from hop-2 (6 slots × $1k) — for a maximum of $33k, subject to pro-rata scaling if any hop is oversubscribed. $33,000 out of a $1.2M base fund is 2.75%.

The actual deterrents to governance concentration remain:

1. **Real capital required at every slot.** $33k via recursive self-invite is the self-fill maximum. Oversubscription scales everyone back pro-rata.
2. **Cap differential.** Hop-0 gets $15k per slot, hop-1 gets $4k per slot, hop-2 gets $1k per slot. The incremental benefit of filling lower hops is real but bounded by the invite tree structure.
3. **Graph visibility.** The full invite graph is public in real time. Multi-hop participation by a single address — including recursive self-invitation — is fully legible.
4. **Governance concentration has low ROI pre-revenue.** Accumulating votes in a protocol with no product yet is not obviously valuable.

Residual risk — a seed concentrating votes at scale — is primarily mitigated by trusted-network seed selection. The mechanism does not make it impossible; it makes it expensive and bounded. Same-address multi-hop is fully visible on-chain; multi-wallet self-fill is visible in graph shape but same-entity control is not provable.

---

## Crowdfund Mechanics

### Timeline

| Phase | Duration |
|---|---|
| Launch team invite window | Days 1–7 (first week only) |
| Seed invite window | Days 1–21 (full 3 weeks) |
| Hop-1 invite window | Days 1–21 (full 3 weeks) |
| Commitment window | Days 1–21 (full 3 weeks) |

The launch team's invite budget (seeds, hop-1, hop-2 placements) may only be issued during the first week. Seeds and hop-1 participants may issue invitations throughout the full three weeks — right up to the commitment deadline. This limits the launch team's ability to issue new invitations with live oversubscription data visible (they can still do so during days 1–7 but not after), while giving seeds and hop-1 invitees maximum flexibility to propagate the network throughout the window. Note: a launch team hop-1 invitee issued on day 7 has the full remaining 14 days to issue their own hop-2 invitations — the launch team's invite window closing does not affect hop-1 invite rights.

### Commitment

- **Currency:** USDC
- **Method:** Commit to escrow contract
- **Multi-hop commits from the same address are permitted.** An address may commit at multiple hop levels and receives ARM allocation from each hop it participates in. This makes entity-level participation unambiguously legible on-chain: a seed who also fills hop-1 and hop-2 slots with the same address is visibly doing so, and receives the ARM from all three pools.
- **Multiple commitments:** A participant may commit multiple times to any hop. Commitments from the same address at the same hop are aggregated into a single running balance, capped at `participation_slots[(address, hop)] × HOP_CAP[hop]` for allocation purposes (each invite to a hop creates a separate participation slot).
- **Visibility:** Address, amount, hop, and invite edges are all public in real time
- **No withdrawals:** Commitments are final once submitted. Participants cannot withdraw USDC before the deadline. The commitment window is a maximum of 3 weeks; participants should only commit amounts they are prepared to lock for that period.

Real-time aggregate statistics show current commitment levels per hop throughout the window, allowing participants to observe oversubscription before committing. **Per-hop totals are the canonical display** — each hop shows the effective demand committed to that hop, regardless of whether any addresses also participate at other hops. An address committed at hop-0 and hop-1 contributes to both hop totals independently. This means the sum of all displayed per-hop totals may exceed the sale size, which is expected and correct — it reflects genuine multi-hop demand that will be allocated and scaled per hop.

### ARM Pre-Load Requirement

**1,800,000 ARM (MAX_SALE) must be present in the crowdfund contract before the commitment window opens.** The ARM token constructor mints 1,800,000 ARM directly to the crowdfund contract address (see ARM_TOKEN.md §3). The crowdfund contract enforces this with a deployment-time flag — the commitment function reverts until ARM is loaded and verified. This ensures participant claim records written at finalization are always backed by sufficient ARM.

Deployment sequence:
1. Deploy `ArmadaCrowdfund` with treasury address, ARM address, timing parameters (ARM token uses CREATE2 precomputed address — see REVENUE_LOCK.md §3)
2. Deploy ARM token — constructor mints 1,800,000 ARM directly to the crowdfund contract
3. Call `loadArm()` to verify balance and set the flag

`loadArm()` may be called before or at the configured open timestamp. If called in advance, the contract is armed but `commit()` reverts until the open timestamp is reached. `addSeed()` and `launchTeamInvite()` are similarly gated to the week-1 window (which begins at the open timestamp) and must not be called before the window is active.

4. Commitment window opens at the configured open timestamp

### Finalization

After the commitment window closes, finalization is permissionless — anyone may call `finalize()` once both conditions are met:

- The commitment deadline has passed, **and**
- `capped_demand` (sum of all effective per-address per-hop balances after caps — an address committed at multiple hops contributes at each hop independently) meets or exceeds the minimum raise of **$1,000,000 USDC**

If the commitment deadline passes without `capped_demand` meeting the minimum raise, the contract does not self-execute. `claimRefund()` derives eligibility from state. A participant may withdraw their full deposited USDC if and only if **any** of these conditions hold:

1. `refundMode == true` — set by the post-allocation check in `finalize()` when net_proceeds < MINIMUM_RAISE
2. `block.timestamp > commitmentDeadline && !finalized && capped_demand < MINIMUM_RAISE` — deadline passed without a qualifying raise. The `capped_demand < MINIMUM_RAISE` clause is critical: without it, a qualifying crowdfund that simply hasn't been finalized yet would incorrectly permit refunds.
3. `cancelled == true` — Security Council invoked `cancel()`

No privileged action or separate function is required — the eligibility check is built into `claimRefund()` itself.

When `finalize()` is called:

> These steps describe settlement outcomes, not mandatory EVM instruction order. Implementation must follow checks-effects-interactions: record all state changes before making external asset transfers.

1. Crowdfund size is determined (base or expanded, based on total capped demand)
2. Per-hop ceilings are calculated
3. Allocations are computed (pro-rata where oversubscribed; addresses at multiple hops receive ARM from each hop independently, accumulated into a single balance)
4. Refund amounts are computed and recorded per address
5. **Post-allocation minimum raise check:** if `net_proceeds < MINIMUM_RAISE ($1,000,000 USDC)`, the contract sets **both `finalized = true` and `refundMode = true`** and stops — no treasury transfer, no ARM allocation recording. Setting `finalized = true` permanently disables further calls to `finalize()` and `cancel()`. Any provisional allocation and refund amounts computed in steps 3–4 are discarded. **In this branch, every participant may call `claimRefund()` to withdraw their full deposited USDC** — not a pro-rata refund, but the complete raw amount they deposited. **`claim()` reverts when `refundMode == true` — no ARM allocations were recorded.** This scenario can occur at base size when hop-0 is oversubscribed and combined later-hop demand doesn't close the gap between hop-0's ceiling ($798k) and the minimum raise ($1M). It cannot occur after expansion, where hop-0's ceiling ($1,197k) already exceeds the minimum raise. This must not revert the transaction — a revert would leave the contract permanently stuck with no path to refunds.
6. **Net USDC proceeds transfer immediately to treasury** — refund amounts are deducted first; only the net proceeds transfer
7. ARM allocations are recorded on-chain for participants to claim

Refunds are pull-based — participants call `claimRefund()` once eligible (see Finalization section for the full three-condition eligibility rule, which includes pre-finalization and cancel-state paths). This avoids gas grief from iterating over all participants in a single transaction. Refunds do not expire.

ARM claim remains a separate participant-initiated step (one transaction per claimant). USDC net proceeds flow to treasury at finalization regardless of whether participants have claimed their ARM or refunds.

Crowdfund tokens are not revenue-gated, but voting power only becomes active once ARM is claimed and delegated. Allocated-but-unclaimed ARM is vote-inert regardless of any governance unlock proposal that passes.

**Governance quiet period: 7 days after finalization.** No governance proposals may be submitted until day 8 post-finalization. This gives the community time to claim, delegate, and orient before governance begins. Any emergency during this window is handled by the Security Council (pause, veto, coordinated response) — it does not require a governance vote. The quiet period is a one-time bootstrapping measure and is itself governable.

**Early-claim governance risk.** Voting power uses Compound-style point-in-time checkpointing — there is no time-weighted average. An address that claims and delegates immediately after finalization has its full balance active with no averaging period. Quorum at proposal creation is measured against only the ARM that has been claimed and delegated by that block. If very few participants have claimed, a single early claimer with 12,000+ ARM could meet proposal threshold and create a vote where quorum is a fraction of the full crowdfund supply. The 48-hour proposal delay and 7-day quiet period are the only buffers. Participants are strongly encouraged to claim and delegate promptly after finalization.

**Claim deadline: 3 years from finalization.** The deadline is fixed and cannot be extended — it is a predeclared term of participation, not a governance parameter. Exact boundary: `claim()` is permitted when `block.timestamp <= finalizationTimestamp + 3 years`; `withdrawUnallocatedArm()` may sweep unclaimed ARM only when `block.timestamp > finalizationTimestamp + 3 years`.

**`withdrawUnallocatedArm()` eligibility — three windows:**
- **Immediately post-finalization:** sweeps `MAX_SALE - allocated_arm` — all ARM not sold to any participant, including (in a base crowdfund) the 600,000 ARM preloaded for potential expansion that was never needed.
- **After the 3-year claim deadline (`block.timestamp > finalizationTimestamp + 3 years`):** sweeps any ARM still in the contract — allocated-but-never-claimed participant allocations.
- **After `cancel()`:** sweeps all preloaded ARM (1,800,000) — the crowdfund is in cancelled state, not finalized, so the function must check `cancelled` as an additional eligibility condition alongside `finalized`.

All three send to the immutable treasury address. The function may be called at any time; it sweeps whatever is currently eligible under the above rules.

**Commitment deadline is contract-enforced.** The contract reverts any commitment transaction submitted after the deadline timestamp. This is a hard contract-level guard, not a UI convention.

Post-finalization, all privileged functions are permanently inactive. `withdrawUnallocatedArm()` is permissionless — anyone may call it, and ARM transfers to the immutable treasury address set at deployment. No privileged role survives finalization.

### Allocation

The allocation algorithm proceeds in four sequential steps, each with a separately bounded budget:

1. **Hop-2 floor reserved first.** 5% of `sale_size` is set aside for hop-2 before hop-0 or hop-1 are allocated. Hop-0 and hop-1 compete only for the remaining `available` pool.

2. **Hop-0 allocated from `available`.** Demand is served up to `hop0_ceiling`. Any unused hop-0 capacity increases hop-1's effective ceiling.

3. **Hop-1 allocated from remaining available.** Hop-1's effective ceiling = `min(hop1_ceiling + hop0_leftover, remaining_available)`. Bounded by what remains after hop-0.

4. **Hop-2 allocated from floor + hop-1 leftover.** Hop-2's effective ceiling = `hop2_floor + hop1_leftover`. The floor is always available regardless of earlier hop demand. Any capacity hop-2 doesn't absorb becomes unsold ARM, recoverable via `withdrawUnallocatedArm()`.

Oversubscription at any hop is resolved pro-rata within that hop's effective ceiling, bounded by per-address per-hop caps (where the cap equals `participation_slots[(address, hop)] × HOP_CAP[hop]`). An address committed at multiple hops receives ARM allocation from each hop independently. See appendix for full pseudocode.

`capped_demand` — the sum of all aggregated per-address per-hop balances after caps — is the canonical demand variable used for the minimum raise check, the expansion trigger, and per-hop statistics. An address committed at multiple hops contributes to `capped_demand` at each hop independently.

### Rollover

Rollover is integrated into the allocation steps rather than a separate pass:

- Hop-0 leftover increases hop-1's effective ceiling (bounded by what remains after hop-0)
- Hop-1 leftover increases hop-2's effective ceiling (stacks on top of the guaranteed floor)
- Hop-2 leftover becomes unsold ARM, recoverable via `withdrawUnallocatedArm()`

Leftover never rolls back to earlier hops.

### Elastic Expansion

The crowdfund expands if `capped_demand` reaches or exceeds the expansion trigger.

```python
if capped_demand >= EXPANSION_TRIGGER:  # $1.5M USDC capped demand
    sale_size = MAX_SALE  # 1,800,000 ARM ($1.8M, 15% of supply)
else:
    sale_size = BASE_SALE  # 1,200,000 ARM ($1.2M, 10% of supply)
```

Expansion is binary: either base ($1.2M) or max ($1.8M). If capped demand clears $1.5M, supply expands to maximum. Oversubscribed hops are allocated pro-rata up to their effective ceiling; the difference between a participant's commitment and their allocation is refunded.

### Refunds

```
refund_usdc = total_deposited_usdc - (allocation_arm × price)
```

Refunds are pull-based — participants call `claimRefund()` once eligible. See Finalization section for the full eligibility conditions.

---

## Emergency Cancel

The crowdfund contract includes an emergency cancel mechanism for catastrophic pre-finalization scenarios: a critical contract bug, a regulatory injunction, or another event that would make proceeding harmful to participants.

**Authority:** Security Council (3-of-5 multisig — see GOVERNANCE.md)

**Process:**
1. Security Council invokes `cancel()` — takes effect immediately
2. Contract enters refund-only mode: all committed USDC becomes withdrawable by participants via `claimRefund()`. No further privileged action required — participants pull their own refunds.

**`cancel()` sets a permanent cancelled state.** After `cancel()` is invoked: `finalize()` permanently reverts (the cancelled flag is checked at entry); `commit()` permanently reverts — no new USDC may be deposited; participants may call `claimRefund()` to withdraw their full deposited USDC; and all preloaded ARM (1,800,000) becomes immediately sweepable to the immutable treasury address via `withdrawUnallocatedArm()`. The cancelled state is irreversible.

**Constraints:**
- Only callable before `finalize()` has been executed
- Once finalized, `cancel()` permanently reverts — no post-finalization admin leverage
- Refund mode is automatic and unconditional; the Security Council has no discretion over individual refunds

**Rationale:** Cancel takes effect immediately because a genuine emergency (active exploit, regulatory injunction) requires speed — a delay serves the attacker, not participants. Abuse protection comes from Security Council composition (not purely team-controlled), automatic unconditional refunds (no discretion over destination), and the pre-finalization-only restriction. The on-chain transaction is public the moment it is submitted.

---

## Governance

Crowdfund tokens are not revenue-gated. Voting power becomes active once ARM is claimed and delegated — unclaimed ARM is vote-inert. Team and airdrop tokens gain voting power proportional to revenue-milestone unlocks — at $0 protocol revenue, only crowdfund participants who have claimed and delegated vote.

ARM must be delegated to vote. Delegation is designated at claim time as a mandatory parameter.

Standard proposals take approximately 11 days (48h delay + 7d vote + 48h execution). High-impact proposals (fee changes, treasury >5%, governance parameters, steward elections) take approximately 18 days (48h + 14d + 7d). Bond: 1,000 ARM, returned after lock period. Proposal threshold: 12,000 ARM.

Crowdfund tokens become transferable via governance proposal. There are no predeclared conditions — holders decide when.

Team and airdrop tokens cannot be unlocked by governance. They unlock only via revenue milestones.

Treasury tokens (65–70%) are controlled by governance but do not vote.

See GOVERNANCE.md for full specification: voting mechanics, delegation, quorum, proposal lifecycle, Treasury Steward, Security Council, and wind-down.

---

## Invite Graph Visibility

The full invite graph is public throughout the crowdfund — inviter, invitee, and hop level are visible from the moment an invite is issued. There is no post-finalization reveal step.

Participants are joining a trust network, and that network's shape should be legible to everyone in it. Full visibility during the crowdfund also reinforces the governance bootstrapping framing: you can see who you're building the network with before committing.

---

## Wind-Down

### Automatic wind-down

The protocol triggers an automatic wind-down if the revenue threshold is not reached by the deadline. Both parameters are governable with the following defaults:

| Parameter | Default | Governable |
|---|---|---|
| Revenue threshold | $10,000 cumulative | Yes |
| Deadline | December 31, 2026 | Yes |

The intent is to give the protocol approximately 6 months after mainnet launch to demonstrate traction. December 31, 2026 is the concrete implementation default and is calibrated to that intent based on the current launch timeline. If launch slips materially, governance should update the deadline before it becomes binding.

If cumulative protocol revenue does not reach the threshold by the deadline, wind-down initiates automatically — no governance vote required. Governance may adjust either parameter before the deadline, or vote to wind down early at any time.

### Wind-down sequence

1. Wind-down triggers (automatic or governance vote). `triggerWindDown()` is permissionless — anyone can call it when conditions are met.
2. ARM transfers are automatically enabled (`setTransferable(true)` called as part of the wind-down transaction)
3. Shielded pool enters withdraw-only mode (immediate — `shield()` and shielded `transfer()` disabled; `unshield()` always available indefinitely with no deadline)
4. Governance ends permanently — no new proposals, no steward proposals, no parameter changes
5. Non-ARM treasury assets are swept to the redemption contract via permissionless `sweepToken(address)` calls (one per token; `sweepToken(ARM)` reverts — treasury ARM stays locked permanently)
6. ARM holders deposit ARM into the redemption contract and receive their pro-rata share of non-ARM treasury assets. No snapshot, no deadline — deposit whenever you want. See GOVERNANCE.md §Wind-Down for the full redemption mechanism.
7. Team and airdrop tokens have no treasury claim unless released via revenue milestones before wind-down. Unreleased ARM stays in the revenue-lock contract permanently.

**Unclaimed ARM after wind-down.** Participants may claim their ARM at any time after wind-down (subject to the 3-year deadline). Once claimed, they can deposit it into the redemption contract to receive their treasury share. The 3-year ARM claim deadline continues to apply. Unclaimed ARM in the crowdfund contract is excluded from the redemption denominator, so it does not dilute redeemers.

Those who paid have priority over those who received tokens at zero cost basis. Locked tokens only unlock if protocol earns revenue — if the protocol failed before earning revenue, those tokens stay locked and have no claim on remaining assets.

---

## Risk Acknowledgments

- **Seed quality is the critical dependency.** The allocation mechanism is algorithmic, but the network's governance quality depends entirely on the launch team's seed selection. Trusted-network seed selection is the primary defence against governance concentration and Sybil attacks — the contract mechanism does not substitute for it. Maximum single-entity capture from one subtree is $33k (2.75% of base fund); an address invited by multiple seeds accumulates beyond this. See Self-Filling section.
- **At base size, hop-0 alone cannot meet the minimum raise.** Hop-0's ceiling ($798k at base fund) is below the minimum raise ($1M). However, the same addresses filling hop-0 can also commit at hop-1 and hop-2 — those contributions count independently. For example, 67 seeds each committing $15k at hop-0 and $4k at hop-1 produces $1.273M capped_demand (base sale) and ~$1.066M net_proceeds — sufficient to finalize. After expansion (capped_demand ≥ $1.5M), hop-0's ceiling rises to $1,197k and hop-0 alone can succeed.
- **Early governance proposals may pass on thin quorum.** Voting power activates immediately upon claim and delegation with no averaging period. The first proposals after the 7-day quiet period may be created when only a small fraction of participants have claimed. Participants are strongly encouraged to claim and delegate promptly after finalization.
- **Governance capture is possible.** Whitelist is the mitigation, not a guarantee.
- **Propagation may fail.** Unused supply flows to treasury.
- **This is a trusted-network crowdfund.** Not designed for viral distribution.
- **Tokens are non-transferable at launch by design.** Transferability prioritizes governance participation during the formative phase. Token transfers can be unlocked by governance proposal.
- **Automatic wind-down is a real risk.** If the protocol does not reach $10k revenue by December 31, 2026, wind-down initiates automatically.

---

## Summary of Design Decisions

| Decision | Resolution | Rationale |
|---|---|---|
| Treasury voting | Treasury does not vote | Avoids control ambiguity |
| Team/airdrop voting | Revenue-gated | Only people who paid decide until protocol proves itself |
| Steward voting eligibility | Voting-eligible ARM only | Treasury decisions by economic stakeholders |
| Invite graph | Fully public throughout the crowdfund | No post-finalization reveal step. Participants are joining a trust network — its shape should be legible from the start. Simplifies implementation; reinforces governance bootstrapping framing over token allocation framing |
| Enactment delay | 48 hours standard; 7 days extended | Short delay is honest — without locking, market prices outcomes during voting anyway. Extended delay gives Security Council a meaningful veto window on high-impact proposals. |
| Elastic expansion | Binary to 1.8M ARM | Accommodates demand without distorting any future fundraising |
| Team allocation control | Single shared revenue-lock contract with per-beneficiary allocations; Knowable Safe is a beneficiary like any other team member | Each team member governs independently; Knowable Safe handles future contributors off-chain after release and global transfer unlock |
| Pre-crowdfund distribution | Team + airdrop distributed before the crowdfund opens | Full transparency for funders |
| Recurring payments | Not supported in v1 | Steward submits monthly batches instead |
| Hop allocation model | Demand-driven ceilings, not fixed reserves | Capacity follows actual demand; no hop is guaranteed allocation it didn't earn; unused capacity rolls forward |
| Overlapping ceilings | Hop-0 and hop-1 sum to 115% of available (70/45); hop-2 has no fixed ceiling | Hop-0 and hop-1 can absorb more if the other underperforms. Hop-2's effective ceiling is floor + hop-1 leftover — it can grow arbitrarily large if earlier hops are empty, preserving full rollover flexibility |
| Hop-2 hard floor | 5% of the fund reserved — hop-2 has first claim on this capacity | Hop-2 is governance breadth and ecosystem signal. The reserved capacity ensures hop-2 can always absorb up to the floor amount regardless of earlier hop demand. Actual receipt depends on hop-2 demand. |
| Multi-hop commits from single address | Permitted; receives ARM allocation at each hop independently | $33k per-subtree ceiling (2.75% of base raise). An address invited by multiple seeds accumulates across all subtrees — no global cap, but all participation is visible in the graph. Concentration is legible, not preventable. |
| Invite window | Launch team: days 1–7 only; Seeds and hop-1: full 3 weeks | Narrows launch team discretion to week 1 — after that, no new seeds can be added and no new launch team invites can be issued. Live oversubscription data is visible during days 1–7, so this reduces but does not eliminate data-informed invite issuance. Seeds and hop-1 invitees retain full flexibility throughout. |
| Rolling seed additions | Launch team may add seeds during week 1 only, up to 150-seed cap | Constrained to launch team's invite window. Once added, seeds may invite throughout the full 3 weeks. |
| Seed cap | 150 seeds maximum | Bounds network to ~1,740 participant slots (nodes); distinct individuals may be fewer under single-address multi-hop |
| Launch team invite budgets | 60 hop-1 + 60 hop-2 invitations, predeclared, week 1 only | Seeds the initial community with allies not connected to any seed. |
| Rollover | Unconditional — leftover always rolls forward | Rollover thresholds dropped as weak sybil protection; seed selection is the real defence. Simpler contract. |
| No commitment withdrawals | Commitments are final once submitted | Eliminates gaming risk and simplifies contract; 3-week maximum lock period is known upfront |
| ARM pre-load requirement | 1,800,000 ARM loaded before window opens | Ensures claim records written at finalization are always backed by sufficient ARM; enforced by contract flag |
| Settlement model | Pull-based for refunds and ARM; push for net proceeds to treasury | `finalize()` records allocations and refund amounts, transfers only net proceeds to treasury. Participants pull refunds via `claimRefund()` and ARM via `claim()`. Avoids gas grief from iterating over all participants in one transaction. |
| Crowdfund finalization | Permissionless — callable by anyone after deadline + minimum raise met | No identified party triggers finalization. If deadline passes without minimum raise, `claimRefund()` derives eligibility from timestamp + state; no separate function or privileged action needed. |
| Crowdfund emergency cancel | Security Council (3-of-5 multisig), immediate effect, pre-finalization only | Speed serves participants in a genuine emergency. Abuse protection from Security Council composition, automatic unconditional refunds, and pre-finalization-only restriction — not a delay. |
| Post-finalization | No privileged roles | All privileged functions permanently inactive after finalization. `withdrawUnallocatedArm` is permissionless — anyone calls it, ARM goes to immutable treasury address. |
| Governance quiet period | 7 days after finalization; no proposals until day 8 | Gives community time to claim, delegate, and orient. Security Council handles any emergency during this window. Governable. |
| ARM claim deadline | 3 years from finalization, fixed | Predeclared term of participation, ungovernable by design |
| Automatic wind-down | Revenue threshold + deadline, both governable | Removes need for governance vote to initiate failure-case wind-down; defaults ($10k by Dec 31 2026) are conservative enough to signal genuine traction |

---

## Contract Interface

### External Functions

| Function | Caller | Parameters | Preconditions | Effects |
|---|---|---|---|---|
| `loadArm()` | Anyone (once) | — | Contract holds ≥ MAX_SALE ARM; not already called | Sets ARM-loaded flag; arms the sale (commitment window opens at configured `openTimestamp`, not at the moment `loadArm()` is called); emits `ArmLoaded` |
| `addSeed(address)` | Launch team | Seed address | ARM loaded; week 1 only; seed count < 150; address not already a seed; not cancelled; not finalized | Adds address as hop-0 node; records edge from ROOT; emits `SeedAdded`; decrements seed budget |
| `launchTeamInvite(invitee, fromHop)` | Launch team | Target address, inviter's hop (0 or 1; invitee joins at `fromHop + 1`) | ARM loaded; week 1 only; budget remaining for target hop; not cancelled; not finalized | Records invite edge from ROOT to invitee at `fromHop + 1`; emits `Invited(ROOT, invitee, fromHop + 1, 0)`; decrements launch team budget |
| `commit(hop, amount)` | Participant | Hop level, USDC amount | ARM loaded; pre-deadline; not cancelled; not finalized; `amount > 0`; address holds at least one participation slot at this hop (hop-0 slots from `SeedAdded`; hop-1/hop-2 slots from `Invited`) | Records commitment; transfers USDC to escrow; emits `Committed` |
| `invite(invitee, fromHop)` | Inviter | Target address, inviter's hop | ARM loaded; pre-deadline; not cancelled; not finalized; caller has available slots at `fromHop` | Records invite edge; creates a new participation slot for invitee at `fromHop + 1` (increases invitee's cap by `HOP_CAP[fromHop + 1]`); emits `Invited(caller, invitee, fromHop + 1, 0)`; decrements inviter's slot count at `fromHop` |
| `commitWithInvite(inviter, fromHop, nonce, deadline, signature, amount)` | Invitee | Inviter address, inviter's hop, nonce, deadline, EIP-712 signature, USDC amount | Valid EIP-712 + EIP-1271 signature; `nonce > 0`; `amount > 0`; `block.timestamp ≤ deadline`; nonce not used/revoked; inviter has available slots at `fromHop`; ARM loaded; pre-deadline; not cancelled; not finalized; USDC approved | Records invite edge + commitment atomically; creates a new participation slot for invitee at `fromHop + 1`; transfers USDC to escrow; emits `Invited(inviter, caller, fromHop + 1, nonce)` + `Committed(caller, fromHop + 1, amount)`; consumes nonce; decrements inviter's slot count |
| `revokeInviteNonce(nonce)` | Inviter | Nonce to revoke | `nonce > 0`; nonce not already consumed or revoked | Marks nonce permanently revoked; emits `InviteNonceRevoked` |
| `finalize()` | Anyone | — | Post-deadline; not finalized; not cancelled; `capped_demand ≥ MINIMUM_RAISE` | Computes allocations; records per-address allocations and refund amounts; sets `finalized = true`; transfers net proceeds to treasury; emits `Finalized`. **Single-transaction mode (preferred):** also emits per-address `Allocated` + `AllocatedHop` events in the same transaction. **Phased mode (fallback):** emits only `Finalized`; settlement events are emitted later via `emitSettlement()`. **RefundMode branch:** if `net_proceeds < MINIMUM_RAISE`, sets both `finalized = true` and `refundMode = true`; no treasury transfer; no allocation/refund recording; emits `Finalized(refundMode=true)` only — no `Allocated` or `AllocatedHop` in either mode. |
| `emitSettlement(startIndex, count)` | Anyone | Start index, batch size | Phased fallback only. `finalized == true`; `refundMode == false`; settlement events not yet fully emitted; `startIndex` must be monotonically advancing (batches are non-overlapping and sequential — no re-emission or out-of-order) | Emits `Allocated` + `AllocatedHop` events for the specified batch. `SettlementComplete` emitted exactly once, automatically when the final batch completes. No state changes beyond tracking emission progress. |
| `claimRefund()` | Participant | — | `refundMode == true` OR (`post-deadline AND !finalized AND capped_demand < MINIMUM_RAISE`) OR `cancelled == true`; refund not already claimed | Transfers refund USDC to caller; emits `RefundClaimed` |
| `claim(delegate)` | Participant | Delegate address | `finalized == true`; `refundMode == false`; `block.timestamp <= finalizationTimestamp + 3 years`; not already claimed | Transfers allocated ARM to caller; records delegation; emits `ArmClaimed` |
| `cancel()` | Security Council | — | Not finalized | Sets `cancelled` permanently; emits `Cancelled` |
| `withdrawUnallocatedArm()` | Anyone | — | `finalized == true` (sweeps unsold) OR post-3yr-deadline (sweeps unclaimed) OR `cancelled == true` (sweeps all 1.8M) | Transfers eligible ARM to immutable treasury address |

**Hop parameter convention:** All invite functions use `fromHop` — the inviter's hop level. The invitee's hop is always `fromHop + 1` and is never passed as a parameter.

---

## Required Contract Events

| Event | Emitted by | Fields | Description |
|---|---|---|---|
| `ArmLoaded` | `loadArm()` | — | ARM pre-load verified; sale armed (commitment window opens at configured `openTimestamp`) |
| `SeedAdded` | `addSeed()` | `address seed` | New hop-0 node added; edge from ROOT |
| `Invited` | `invite()`, `commitWithInvite()`, `launchTeamInvite()` | `address inviter, address invitee, uint8 hop, uint256 nonce` | New invite edge. `hop` is the invitee's hop level. `nonce` is 0 for direct invites, `> 0` for link-based invites — enables UI to track link consumption from events alone. |
| `Committed` | `commit()`, `commitWithInvite()` | `address participant, uint8 hop, uint256 amount` | USDC committed at specific hop; amount is raw deposit (may exceed cap) |
| `InviteNonceRevoked` | `revokeInviteNonce()` | `address inviter, uint256 nonce` | Invite link nonce permanently invalidated |
| `Finalized` | `finalize()` | `uint256 saleSize, uint256 allocatedArm, uint256 netProceeds, bool refundMode` | Settlement completed; `refundMode` flag distinguishes success from refundMode finalization without reading contract storage |
| `Allocated` | `finalize()` or `emitSettlement()` | `address indexed participant, uint256 totalArmAmount, uint256 totalRefundAmount` | Aggregate per-address settlement. One event per participating address. Success path only (not refundMode). In single-tx mode, emitted by `finalize()`; in phased mode, emitted by `emitSettlement()`. |
| `AllocatedHop` | `finalize()` or `emitSettlement()` | `address indexed participant, uint8 indexed hop, uint256 armAmount` | Per-hop ARM allocation. One event per (address, hop) with `armAmount > 0`. Success path only. No refund field — refunds are aggregate (see `Allocated`). Settlement invariant: `sum(AllocatedHop.armAmount) == Allocated.totalArmAmount` per address. |
| `ArmClaimed` | `claim()` | `address participant, uint256 armAmount, address delegate` | ARM transferred; delegation recorded |
| `RefundClaimed` | `claimRefund()` | `address participant, uint256 usdcAmount` | Refund USDC transferred |
| `Cancelled` | `cancel()` | — | Crowdfund permanently cancelled by Security Council |
| `SettlementComplete` | `emitSettlement()` (phased fallback only) | — | All `Allocated` and `AllocatedHop` events emitted. Emitted exactly once, automatically on final batch. Under single-transaction finalization, all settlement events arrive in the same transaction as `Finalized` and this event is not needed. |

**Event design principles:**
- Every state change emits an event. The observer reconstructs full crowdfund state from events alone — no view functions required for the core display.
- `Allocated` and `AllocatedHop` are emitted on the success path only — by `finalize()` in single-transaction mode, or by `emitSettlement()` in phased mode. In refundMode, neither is emitted — the observer derives full-refund eligibility from `Finalized(refundMode=true)` combined with `Committed` totals.
- `AllocatedHop` is only emitted when `armAmount > 0`. Zero-allocation hops produce no event — the UI infers zero from absence.
- `commitWithInvite()` emits both `Invited` and `Committed` in the same transaction; the observer processes them independently.

### Gas Considerations

**Single-transaction finalization is preferred.** On the success path, `finalize()` emits `Allocated` (one per address, ~800 events at max) plus `AllocatedHop` (one per non-zero (address, hop), up to ~1,600 events). At ~1,200 gas per event, that's ~2.9M gas for events alone, on top of allocation computation and storage writes. This should be feasible within a 30M block gas limit but must be verified during implementation.

**Phased finalization is acceptable as a fallback** if single-transaction gas exceeds block limits. In this case, `finalize()` computes and stores allocations in one transaction, and a separate `emitSettlement(startIndex, count)` function emits `Allocated` + `AllocatedHop` events in batches. If phased finalization is implemented:
- `claim()` is available immediately after `finalize()` — the on-chain allocation records are written during `finalize()` regardless of whether settlement events have been emitted. Event emission is a transparency layer, not a gating condition for claims.
- The UI must handle an intermediate state: settlement computed but events not yet fully emitted. Show "Settlement in progress — allocation data arriving" rather than incomplete data as final.

**Settlement mode selection.** The contract must support both paths. `finalize()` always computes and stores allocations and refund amounts. Whether `finalize()` also emits settlement events in the same transaction is determined during development and gas testing — not at runtime by the operator. The selection mechanism (compile-time constant, constructor flag, or automatic gas-based branching) is implementation-defined. Both modes must produce identical economic outcomes — same `Allocated.totalArmAmount`, same `Allocated.totalRefundAmount`, same `AllocatedHop.armAmount` per (address, hop), same unsold ARM. Only event timing differs. `emitSettlement()` should revert if the contract was deployed in single-tx mode (settlement events already emitted by `finalize()`).
- The final `emitSettlement()` batch emits `SettlementComplete()`. The observer treats `Finalized` without `SettlementComplete` as settlement-in-progress.

### EIP-712 Typed Data (Invite Links)

```solidity
bytes32 constant INVITE_TYPEHASH = keccak256(
    "Invite(address inviter,uint8 fromHop,uint256 nonce,uint256 deadline)"
);

bytes32 constant DOMAIN_TYPEHASH = keccak256(
    "EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"
);
```

The contract stores the domain separator at deployment (computed from the four domain fields). Signature verification uses `SignatureChecker.isValidSignatureNow(inviter, digest, signature)` from OpenZeppelin, supporting both EOA and EIP-1271 contract wallets.

Nonces are per-inviter, any unique `uint256 > 0` — nonce 0 is reserved as the sentinel value for direct invites (`invite()` emits `Invited(..., nonce=0)`). `commitWithInvite()` requires `nonce > 0` and reverts otherwise. The contract imposes no ordering requirement on valid nonces. Each nonce can be consumed (by successful redemption) or revoked (by the inviter) exactly once. A mapping `mapping(address => mapping(uint256 => bool))` tracks consumed and revoked nonces.

---

## Appendix: Allocation Pseudocode

The following pseudocode is the **specification-intent** description of the allocation algorithm. It uses simplified arithmetic for readability. The contract implementation must use **integer math throughout**: percentages as basis points (BPS), token amounts in their smallest denomination (USDC: 6 decimals, ARM: 18 decimals), floor rounding on all pro-rata splits (participants may receive fractionally less than their exact share; dust accumulates as unallocated ARM), and no floating-point operations.

**Two clean asset invariants the implementation must satisfy:**
- **USDC:** `net_proceeds + sum(refunds) == sum(total_deposited)` — all deposited USDC is either sent to treasury or returned to participants; none is stranded
- **ARM:** `allocated_arm + unsold_arm == MAX_SALE` — all pre-loaded ARM (always 1,800,000) is either allocated to participants or immediately recoverable as unsold capacity

**Contract preconditions (enforced before `finalize()` is called):**
- An address may commit at multiple hop levels and receives ARM allocation from each independently.
- Per-address cap at any hop = `participation_slots[(address, hop)] × HOP_CAP[hop]`. Slot sources: hop-0 from `SeedAdded`; hop-1/hop-2 from `Invited` events.
- `loadArm()` has been called; contract holds ≥ `MAX_SALE` ARM.
- The commitment deadline has passed and `capped_demand >= MINIMUM_RAISE`.
- After computing allocations, if `net_proceeds < MINIMUM_RAISE`, the contract sets both `finalized = true` and `refundMode = true` without reverting — `finalized = true` permanently disables `finalize()` and `cancel()`; `claimRefund()` activates via the `refundMode` condition. This can occur at base size when hop-0 is oversubscribed and combined later-hop demand doesn't close the gap to $1M. Cannot occur after expansion.

```python
# Constants
TOTAL_SUPPLY      = 12_000_000
BASE_SALE         = 1_200_000       # ARM (= USDC at $1/ARM)
MAX_SALE          = 1_800_000       # ARM
MINIMUM_RAISE     = 1_000_000       # USDC — checked against both capped_demand and net_proceeds
EXPANSION_TRIGGER = 1_500_000       # USDC capped demand threshold
PRICE             = 1               # Specification uses whole-unit amounts for clarity.
# Integer implementation: 1 whole ARM = 1e18 ARM units; 1 USDC = 1e6 USDC units.
# Price ratio: 1e6 USDC units per 1e18 ARM units (i.e. 1e-12 USDC units per ARM unit).
# All token amounts in the contract must use smallest units; this pseudocode uses
# whole units throughout for readability. Do not read PRICE = 1 as a unit conversion.

HOP_CEILING_BPS = {0: 7000, 1: 4500}    # BPS of available pool; hop-2 uses floor+rollover, not a BPS ceiling
HOP2_FLOOR_BPS  = 500                   # BPS of sale_size
HOP_CAP         = {0: 15_000, 1: 4_000, 2: 1_000}  # USDC per slot per hop
# Per-address cap at a given hop = participation_slots[(address, hop)] × HOP_CAP[hop]
# Slot sources by hop:
#   hop-0: count of SeedAdded events where seed=address (always 0 or 1)
#   hop-1/hop-2: count of Invited events where invitee=address and invitee_hop=hop


def allocate(commitments, participation_slots):
    """
    commitments:        list of Commitment(address, hop, usdc_committed)
    participation_slots: dict of (address, hop) -> int (number of slots at that hop)
                         hop-0 slots from SeedAdded; hop-1/hop-2 slots from Invited events
    An address may commit at multiple hops and receives ARM allocation from each.
    Per-address cap at hop = participation_slots[(address, hop)] × HOP_CAP[hop].
    Deposits above the cap are permitted; excess is refunded at settlement.
    """

    # Step 1: Aggregate multiple commitments per (address, hop),
    # capped at participation_slots[(address, hop)] × HOP_CAP[hop].
    aggregated = {}  # (address, hop) -> effective USDC (capped)
    for c in commitments:
        key = (c.address, c.hop)
        slots = participation_slots.get(key, 0)
        cap = slots * HOP_CAP[c.hop]
        aggregated[key] = min(
            aggregated.get(key, 0) + c.usdc_committed,
            cap
        )

    # Step 2: Compute capped_demand and determine sale size.
    # capped_demand = sum across ALL (address, hop) pairs — each hop counted independently.
    capped_demand = sum(aggregated.values())  # USDC
    sale_size = MAX_SALE if capped_demand >= EXPANSION_TRIGGER else BASE_SALE

    # Step 3: Reserve hop-2 floor upfront.
    # Hop-2 has first claim on this capacity; hop-0 and hop-1 compete only for available.
    hop2_floor   = HOP2_FLOOR_BPS * sale_size // 10_000
    available    = sale_size - hop2_floor

    hop0_ceiling = HOP_CEILING_BPS[0] * available // 10_000
    hop1_ceiling = HOP_CEILING_BPS[1] * available // 10_000

    def participants(hop):
        # All addresses committed at this hop, regardless of other hops they joined.
        return {addr: amt for (addr, h), amt in aggregated.items() if h == hop}

    allocations = {}      # address -> total ARM allocated (accumulated across all hops)
    hop_allocations = {}  # (address, hop) -> ARM allocated at this specific hop (for AllocatedHop events)

    # Step 4: Allocate hop-0 from available pool.
    p0 = participants(0)
    d0 = sum(p0.values())
    if d0 <= hop0_ceiling:
        for addr, amt in p0.items():
            arm = amt // PRICE
            allocations[addr] = allocations.get(addr, 0) + arm
            hop_allocations[(addr, 0)] = arm
        hop0_leftover = hop0_ceiling - d0
        eff0 = d0
    else:
        for addr, amt in p0.items():
            arm = amt * hop0_ceiling // d0 // PRICE  # floor round
            allocations[addr] = allocations.get(addr, 0) + arm
            hop_allocations[(addr, 0)] = arm
        hop0_leftover = 0
        eff0 = hop0_ceiling

    remaining_available = available - eff0

    # Step 5: Allocate hop-1 from remaining available pool.
    # Hop-0 leftover increases hop-1's effective ceiling,
    # but hop-1 can never absorb more than remaining_available.
    hop1_eff_ceiling = min(hop1_ceiling + hop0_leftover, remaining_available)
    p1 = participants(1)
    d1 = sum(p1.values())
    if d1 <= hop1_eff_ceiling:
        for addr, amt in p1.items():
            arm = amt // PRICE
            allocations[addr] = allocations.get(addr, 0) + arm
            hop_allocations[(addr, 1)] = arm
        hop1_leftover = hop1_eff_ceiling - d1
        eff1 = d1
    else:
        for addr, amt in p1.items():
            arm = amt * hop1_eff_ceiling // d1 // PRICE  # floor round
            allocations[addr] = allocations.get(addr, 0) + arm
            hop_allocations[(addr, 1)] = arm
        hop1_leftover = 0
        eff1 = hop1_eff_ceiling

    # Step 6: Allocate hop-2 from floor + any hop-1 leftover.
    # Floor capacity is always available (reserved in Step 3).
    hop2_eff_ceiling = hop2_floor + hop1_leftover
    p2 = participants(2)
    d2 = sum(p2.values())
    if d2 <= hop2_eff_ceiling:
        for addr, amt in p2.items():
            arm = amt // PRICE
            allocations[addr] = allocations.get(addr, 0) + arm
            hop_allocations[(addr, 2)] = arm
        eff2 = d2
    else:
        for addr, amt in p2.items():
            arm = amt * hop2_eff_ceiling // d2 // PRICE  # floor round
            allocations[addr] = allocations.get(addr, 0) + arm
            hop_allocations[(addr, 2)] = arm
        eff2 = hop2_eff_ceiling

    # Step 7: Compute USDC and ARM accounting.
    #
    # USDC accounting:
    #   net_proceeds  = USDC actually sold = ARM allocated × PRICE (NOT eff0+eff1+eff2,
    #                   which overcounts dust from floor rounding)
    #   refunds       = deposited USDC not allocated → claimable via claimRefund()
    #   invariant:    net_proceeds + sum(refunds) == sum(total_deposited)
    #
    # ARM accounting:
    #   allocated_arm = ARM owed to participants → claimable via claim()
    #   unsold_arm    = ARM never allocated → claimable via withdrawUnallocatedArm() immediately
    #   invariant:    allocated_arm + unsold_arm == MAX_SALE
    #   (Allocated-but-unclaimed ARM after 3-year deadline also swept by withdrawUnallocatedArm())

    allocated_arm = sum(allocations.values())                  # ARM
    net_proceeds  = allocated_arm * PRICE                      # USDC (floor-rounding dust stays in contract as unsold_arm)
    unsold_arm    = MAX_SALE - allocated_arm                   # ARM — always uses MAX_SALE, not sale_size
    # 1,800,000 ARM is preloaded regardless of whether the sale expands.
    # In a base sale, the 600,000 expansion reserve is also immediately sweepable.

    # Post-allocation minimum raise check.
    # IMPORTANT: must NOT revert. If net_proceeds < MINIMUM_RAISE, set refundMode = true.
    # All provisional allocation/refund calculations above are DISCARDED.
    # claimRefund() in this branch returns each address's full raw deposited USDC.
    if net_proceeds < MINIMUM_RAISE:
        total_deposited = {}
        for c in commitments:
            total_deposited[c.address] = total_deposited.get(c.address, 0) + c.usdc_committed
        # refundMode = true: each address claims total_deposited[address], no ARM allocated
        full_refunds = {addr: deposited for addr, deposited in total_deposited.items()}
        # Verify: sum(full_refunds) == sum(total_deposited); no ARM allocated
        assert sum(full_refunds.values()) == sum(total_deposited.values())
        # No Allocated or AllocatedHop events emitted in refundMode.
        return None, None, full_refunds, MAX_SALE, 0

    total_deposited = {}
    for c in commitments:
        total_deposited[c.address] = total_deposited.get(c.address, 0) + c.usdc_committed

    refunds = {}
    for addr, deposited in total_deposited.items():
        allocated_usdc = allocations.get(addr, 0) * PRICE
        refunds[addr] = deposited - allocated_usdc  # always >= 0

    # Verify invariants (implementation must enforce these):
    assert net_proceeds + sum(refunds.values()) == sum(total_deposited.values())  # USDC
    assert allocated_arm + unsold_arm == MAX_SALE                                  # ARM
    # Settlement invariant: per-hop ARM sums to aggregate ARM for each address.
    # This is the property tested by: sum(AllocatedHop.armAmount) == Allocated.totalArmAmount
    for addr in allocations:
        per_hop_sum = sum(arm for (a, h), arm in hop_allocations.items() if a == addr)
        assert per_hop_sum == allocations[addr]

    # Return values map to events:
    #   allocations     -> Allocated(address, totalArmAmount, totalRefundAmount)
    #   hop_allocations -> AllocatedHop(address, hop, armAmount) for each entry with armAmount > 0
    #   refunds         -> included in Allocated event's totalRefundAmount field
    return allocations, hop_allocations, refunds, unsold_arm, net_proceeds
```

**Key properties the implementation must preserve:**

| Property | Guarantee |
|---|---|
| Hop-2 reserved capacity | Hop-2 always has first claim on `HOP2_FLOOR_BPS × sale_size`; actual receipt depends on hop-2 demand |
| USDC conservation | `net_proceeds + sum(refunds) == sum(total_deposited)` |
| ARM conservation | `allocated_arm + unsold_arm == MAX_SALE` (1,800,000 always preloaded; base-sale expansion reserve is immediately sweepable) |
| Minimum raise | If `net_proceeds < MINIMUM_RAISE` after allocation, contract sets `refundMode = true` (no revert — state must be preserved for `claimRefund()` to work) |
| Integer rounding | All pro-rata splits floor-rounded; dust accumulates in `unsold_arm` |

**Asset flows post-finalization:**

| Asset | When | Mechanism |
|---|---|---|
| Net USDC proceeds | At finalization | Push to immutable treasury address |
| Excess USDC (refunds) | After finalization, no expiry | Pull: participant calls `claimRefund()` |
| Unsold ARM | After finalization | Pull: anyone calls `withdrawUnallocatedArm()` |
| Allocated-but-unclaimed ARM | After 3-year claim deadline | Pull: anyone calls `withdrawUnallocatedArm()` (sweeps remaining ARM balance at that point) |