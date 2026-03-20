# Armada CROWDFUND.md — Deep Review Brief

## What this is

Armada is a privacy infrastructure protocol for USDC on EVM chains, inheriting Railgun's audited ZK circuits. This brief gives a fresh reviewer everything needed to independently review CROWDFUND.md for the contract implementation and audit.

Treat it as potentially wrong. Your job is to find anything that would change implementation behavior — inconsistencies, underspecifications, or active misstatements.

---

## Key design decisions (intentional — do not re-litigate these)

- **Multi-slot allocation model.** Each invite to an `(address, hop)` creates a new participation slot. Per-address cap at any hop = `participation_slots[(address, hop)] × HOP_CAP[hop]`. Slot sources: hop-0 from `SeedAdded`; hop-1/hop-2 from `Invited` events. This means duplicate invites to the same address at the same hop have real economic value — they increase the invitee's cap. Rationale: any per-address limit that can be bypassed with a second wallet should not exist, because the bypass makes the distribution less legible.
- **Multi-hop single-address commits are permitted and receive ARM from every hop.** An address may commit at hop-0, hop-1, and hop-2 simultaneously. Maximum per-subtree entity capture is $33k (1 hop-0 slot + 3 hop-1 slots + 6 hop-2 slots = 10 positions, all via one address using recursive self-invitation). An address invited by multiple independent seeds accumulates across subtrees with no global cap — all participation is visible in the graph.
- **`capped_demand` counts ALL (address, hop) pairs.** An address committed at hop-0 and hop-1 contributes to both. There is no primary-hop or lowest-hop deduplication. This is the canonical demand variable for minimum raise check, expansion trigger, and live stats.
- **Two invite paths: link-based (primary) and direct (fallback).** Link path uses EIP-712 signed bearer credentials + `commitWithInvite()` for atomic invite+commit. Direct path uses `invite(invitee, fromHop)` for addressed placements. Both produce the same `Invited` event and graph edge. All invite functions use `fromHop` (inviter's hop); invitee's hop is always `fromHop + 1`.
- **Invite link nonces: any `uint256 > 0`.** Nonce 0 is reserved as the sentinel for direct invites (`invite()` emits `Invited(..., nonce=0)`). `commitWithInvite()` requires `nonce > 0`. Links have 5-day default expiry via `deadline` in the signed payload. Revocable via `revokeInviteNonce(nonce)`.
- **EIP-1271 support for invite signatures.** Contract uses OpenZeppelin's `SignatureChecker.isValidSignatureNow()` — supports both EOA and smart contract wallet inviters.
- **Dual settlement events.** `Allocated(address, totalArmAmount, totalRefundAmount)` for aggregate per-address settlement + `AllocatedHop(address, hop, armAmount)` for per-hop transparency. Settlement invariant: `sum(AllocatedHop.armAmount) == Allocated.totalArmAmount` per address. Neither emitted in refundMode.
- **Single-tx finalization preferred; phased fallback acceptable.** In phased mode, `finalize()` emits only `Finalized`; `emitSettlement(startIndex, count)` emits `Allocated` + `AllocatedHop` in monotonic batches; final batch emits `SettlementComplete`. `claim()` is available immediately after `finalize()` regardless of event emission.
- **Hop-2 has no enforced ceiling — only a floor.** `HOP_CEILING_BPS` contains no entry for hop-2. Its effective ceiling is `hop2_floor + hop1_leftover`, which can range from $60k to the full sale size.
- **Over-cap deposits are permitted and refunded** (not reverted at submission).
- **Refunds never expire.** `claimRefund()` has no deadline.
- **ARM claim deadline: 3 years, ungovernable.**
- **Rollover is unconditional.** Leftover always flows forward — no participation thresholds.
- **Finalization is permissionless** after deadline + capped_demand ≥ MINIMUM_RAISE.
- **`refundMode` sets `finalized = true` AND `refundMode = true`.** Permanently disables `finalize()` and `cancel()`.
- **Emergency cancel is immediate, pre-finalization only.** Security Council authority.
- **Seeds invite throughout all 3 weeks.** Only the launch team's budget is limited to week 1.
- **Hop-1 participants may invite throughout all 3 weeks** (same as seeds).
- **Governance quiet period: 7 days post-finalization.** No proposals until day 8.
- **Quorum floor: 100,000 ARM.** `quorum = max(20% circulating, 100,000 ARM)`.
- **Wind-down: all allocated crowdfund ARM counts** (claimed + unclaimed) for treasury distribution. ARM transfers not frozen post-wind-down.

---

## The allocation algorithm

**Two clean asset invariants the implementation must satisfy:**
- **USDC:** `net_proceeds + sum(refunds) == sum(total_deposited)`
- **ARM:** `allocated_arm + unsold_arm == MAX_SALE` (1,800,000 always preloaded)

**Key parameters:**
```python
BASE_SALE         = 1_200_000   # ARM
MAX_SALE          = 1_800_000   # ARM
MINIMUM_RAISE     = 1_000_000   # USDC
EXPANSION_TRIGGER = 1_500_000   # USDC capped_demand (all hops, all addresses)
PRICE             = 1           # USDC per ARM (whole units; see decimal note in appendix)

HOP_CEILING_BPS = {0: 7000, 1: 4500}  # of available pool (hop-2 has no BPS entry)
HOP2_FLOOR_BPS  = 500                  # of sale_size
HOP_CAP         = {0: 15_000, 1: 4_000, 2: 1_000}  # USDC per slot per hop
# Per-address cap = participation_slots[(address, hop)] × HOP_CAP[hop]
# Slot sources: hop-0 from SeedAdded; hop-1/hop-2 from Invited events
```

**Algorithm summary:**
1. Aggregate commits per `(address, hop)`, cap at `participation_slots[(address, hop)] × HOP_CAP[hop]`
2. `capped_demand = sum(aggregated.values())` — all (address, hop) pairs, no deduplication
3. `sale_size = MAX_SALE if capped_demand >= 1_500_000 else BASE_SALE`
4. Reserve `hop2_floor = 500 BPS × sale_size`; `available = sale_size - hop2_floor`
5. Allocate hop-0 from `available` up to `hop0_ceiling = 70% × available`; accumulate into `allocations[addr]`
6. Allocate hop-1 from remaining available, effective ceiling = `min(hop1_ceiling + hop0_leftover, remaining_available)`; accumulate into `allocations[addr]`
7. Allocate hop-2 from `hop2_floor + hop1_leftover`; accumulate into `allocations[addr]`
8. `net_proceeds = sum(allocations.values()) × PRICE` (not eff0+eff1+eff2 — that overcounts floor-rounding dust)
9. `unsold_arm = MAX_SALE - sum(allocations.values())`
10. If `net_proceeds < MINIMUM_RAISE`: set `finalized = true` AND `refundMode = true`; no treasury transfer; `claimRefund()` returns full raw deposits; `claim()` reverts
11. `refunds[addr] = total_deposited[addr] - allocations.get(addr, 0) × PRICE`

**`claimRefund()` eligibility — all three conditions:**
1. `refundMode == true`
2. `block.timestamp > commitmentDeadline AND !finalized AND capped_demand < MINIMUM_RAISE`
3. `cancelled == true`

**`withdrawUnallocatedArm()` — three eligibility windows:**
- Post-finalization: sweeps `MAX_SALE - allocated_arm` (unsold + unused expansion reserve)
- Post-3yr-claim-deadline: sweeps remaining ARM balance (allocated-but-unclaimed)
- Post-cancel: sweeps all 1,800,000 ARM (must check `cancelled`, not just `finalized`)

**Cancel state — all effects:**
- `finalize()` permanently reverts
- `commit()` permanently reverts — no new USDC may be deposited
- `claimRefund()` available for full deposits
- `withdrawUnallocatedArm()` available for all 1,800,000 ARM
- Irreversible

---

## Critical structural property

At base size, hop-0's ceiling ($798k) < MINIMUM_RAISE ($1M). Hop-0 commits alone cannot produce sufficient net_proceeds. However, the same addresses at hop-0 can also commit at hop-1 and hop-2 — those contributions count independently. 67 seeds each at hop-0 ($15k) + hop-1 ($4k) produces capped_demand = $1.273M and net_proceeds ≈ $1.066M — sufficient to finalize at base size.

After expansion (capped_demand ≥ $1.5M), hop-0 ceiling rises to $1,197k and hop-0 alone can succeed.

**Expansion is now easier to trigger** under the multi-slot all-hops model. 46 seeds each filling their full subtree ($33k each via recursive self-invitation) produces capped_demand = 46 × $33k = $1.518M — expansion triggers. Without recursive self-invitation: 75 seeds each at hop-0 ($15k) + one hop-1 slot ($4k) + one hop-2 slot ($1k) = $20k each produces capped_demand = 75 × $20k = $1.5M (expansion trigger).

---

## Known history — issues found and fixed in prior rounds

Do not re-find these; verify they are cleanly resolved:

1. `net_proceeds = sale_size - refunds` → wrong; fixed to `allocated_arm × PRICE`
2. ARM invariant used `sale_size` → fixed to `MAX_SALE`
3. `refundMode` used assert/revert → fixed to flag-setting, non-reverting
4. `treasury_leftover` was USDC → renamed `unsold_arm`, separated asset types
5. Hop-2 in same budget counter as hop-0/1 → fixed to separate upfront reservation
6. `reallocate_hop()` didn't update `remaining_budget` → entire rollover pass replaced with integrated per-step leftover
7. `HOP_CEILING_BPS[2]` was dead code → removed
8. "Automatic refund-only mode" → clarified as `claimRefund()` derived eligibility
9. "Finalize reverts to refund-only" → impossible; fixed to flag-setting
10. Hop-2 phantom 10% ceiling in hop structure table → removed; hop-2 has no enforced ceiling
11. Launch-team graph root undefined → defined as `(launch_team_address, ROOT)` sentinel node
12. Hop-1 invite window undefined → days 1–21 (same as seeds)
13. "Primary hop wins" / "lower hop only" model → **reversed**: all hops receive allocation independently
14. `capped_demand` was primary-hop-only → **reversed**: all (address, hop) pairs counted
15. `refundMode` branch didn't set `finalized = true` → fixed; both flags set simultaneously
16. `cancel()` didn't specify `commit()` reverts → fixed
17. "After finalization" in refund prose → fixed to "once eligible"
18. Hop-2 phantom 10% ceiling in reserves → removed
19. Graph root undefined → ROOT sentinel
20. $20k → $33k entity ceiling correction for multi-slot model
21. Stale `capped_demand` definition at finalization entry → fixed
22. Observer node model singular `invited_by: address` → changed to `address[]`
23. Nonce 0 ambiguity → reserved for direct invites; link nonces require `> 0`
24. Seed eligibility from `SeedAdded` not `Invited` → `commit()` precondition updated
25. Pseudo-slot UI → replaced with outstanding-links model in committer
26. Per-hop settlement impossible from aggregate-only events → dual event model (`Allocated` + `AllocatedHop`)
27. `fromHop`/`toHop` parameter inconsistency → standardized on `fromHop`
28. ARM loaded missing from invite preconditions → added to `invite()`, `addSeed()`, `launchTeamInvite()`
29. `emitSettlement` not in function table → added with monotonic-batch precondition
30. `revokeInviteNonce(0)` should revert → added `nonce > 0` check
31. `Allocated`/`AllocatedHop` emitters didn't include `emitSettlement()` → fixed in events table
32. Observer missing `SettlementComplete` in consumed events → added
33. `finalize()` effects didn't mention refund recording → added "records refund amounts"
34. Pseudocode didn't track per-hop allocations → added `hop_allocations` dict for `AllocatedHop` events
35. Pseudocode `invite_slots` didn't include `SeedAdded` for hop-0 → renamed to `participation_slots` with explicit slot sources

---

## What to look for

### 1. Internal consistency
- Does every description of `capped_demand` say "all hops, no deduplication"?
- Does every description of multi-hop allocation say "ARM received at each hop"?
- Does the prose match the pseudocode exactly?

### 2. State machine completeness

For each external function, verify: allowed states, revert conditions, state changes, asset movements.

| Function | Allowed states | Reverts if | State change | Asset movement |
|---|---|---|---|---|
| `loadArm()` | Pre-commitment, not yet called | Already called; balance < MAX_SALE | Sets ARM-loaded flag; emits `ArmLoaded` | — |
| `addSeed(address)` | ARM loaded, week 1, not cancelled/finalized | Not week 1; seed cap reached; not ARM loaded | Adds hop-0 node; emits `SeedAdded` | — |
| `launchTeamInvite(invitee, fromHop)` | ARM loaded, week 1, not cancelled/finalized | Not week 1; budget exhausted; not ARM loaded | Creates participation slot for invitee at `fromHop + 1`; emits `Invited(..., nonce=0)` | — |
| `invite(invitee, fromHop)` | ARM loaded, pre-deadline, not cancelled/finalized | No available slots at `fromHop`; not ARM loaded | Creates participation slot for invitee at `fromHop + 1`; emits `Invited(..., nonce=0)` | — |
| `commitWithInvite(inviter, fromHop, nonce, deadline, signature, amount)` | ARM loaded, pre-deadline, not cancelled/finalized | Invalid/expired signature; nonce ≤ 0 or consumed/revoked; inviter out of slots; not ARM loaded | Creates slot + records commitment atomically; emits `Invited(..., nonce)` + `Committed` | USDC in |
| `revokeInviteNonce(nonce)` | Any (inviter is msg.sender) | nonce == 0; already consumed/revoked | Marks nonce permanently revoked; emits `InviteNonceRevoked` | — |
| `commit(hop, amount)` | ARM loaded, pre-deadline, not cancelled/finalized | No participation slot at this hop; not ARM loaded | Records commitment; emits `Committed` | USDC in |
| `finalize()` | Post-deadline, not finalized, not cancelled, capped_demand ≥ MIN | Already finalized; cancelled; capped_demand < MIN | Sets `finalized = true`; records allocations/refunds; AND sets `refundMode = true` if net_proceeds < MIN. Emits `Finalized` (+ `Allocated`/`AllocatedHop` in single-tx mode) | Net proceeds out to treasury (success path only) |
| `emitSettlement(startIndex, count)` | Finalized, not refundMode, phased fallback only | Not finalized; refundMode; out-of-order batch | Emits `Allocated` + `AllocatedHop` for batch; `SettlementComplete` on final batch | — |
| `claimRefund()` | refundMode OR (post-deadline AND !finalized AND capped_demand < MIN) OR cancelled | None of those conditions; already claimed | — | USDC out to participant |
| `claim(delegate)` | Finalized AND NOT refundMode, within 3-year deadline | Not finalized; refundMode == true; expired; already claimed | Records ARM claimed; records delegation; emits `ArmClaimed` | ARM out to participant |
| `cancel()` | Pre-finalization | Already finalized | Sets cancelled permanently; emits `Cancelled` | — |
| `withdrawUnallocatedArm()` | Finalized (unsold) OR post-3yr (unclaimed) OR cancelled (all 1.8M) | None of those conditions | — | ARM out to treasury |

### 3. Invite mechanism and graph semantics
- Is the graph defined over `(address, hop)` nodes consistently?
- Does the launch-team sentinel node `(launch_team_address, ROOT)` appear and is it clear it makes no commitment?
- Does same-address multi-hop create self-loops correctly?
- Are hop-1 invite rights specified as days 1–21?
- Do live stats display per-hop totals (each hop independently, no deduplication)?
- Does each `Invited` event create a new participation slot (multi-slot model)?
- Is the `fromHop` convention used consistently for all invite functions?
- Are nonce semantics correct: `> 0` for links, `0` for direct invites, `revokeInviteNonce(0)` reverts?
- Is the EIP-712 typed data structure complete: `inviter`, `fromHop`, `nonce`, `deadline`?
- Does `commitWithInvite()` bundle invite + commit atomically?
- Is signature verification via `SignatureChecker` (EOA + EIP-1271)?

### 4. Contract events
- Are `Allocated` and `AllocatedHop` emitted by `finalize()` (single-tx) or `emitSettlement()` (phased)?
- Is `SettlementComplete` emitted exactly once on the final `emitSettlement()` batch?
- Does `Invited` include `nonce` (0 for direct, > 0 for link-based)?
- Is the settlement invariant stated: `sum(AllocatedHop.armAmount) == Allocated.totalArmAmount`?
- Are `Allocated`/`AllocatedHop` absent in refundMode?
- Is `AllocatedHop` only emitted when `armAmount > 0`?

### 5. Economic consequences of the multi-slot all-hops model
- Is it disclosed that expansion is now easier to trigger?
- Is it disclosed that self-filling lower hops now has direct economic value ($4k + $1k additional allocation)?
- Is it disclosed that each invite to a hop creates real economic value (increases the invitee's cap)?
- Is the $33k max-subtree-capture arithmetic correct for multi-slot (1×$15k + 3×$4k + 6×$1k)?
- Is the refundMode scenario described accurately (much harder to reach than before)?

### 6. Precision for implementation
- Would an implementer reading only one section miss something from another?
- Are all `capped_demand` references consistent with the all-hops definition?
- Does the pseudocode's `participation_slots` parameter match the slot-source definitions?
- Does the pseudocode return per-hop allocations (for `AllocatedHop` events)?
- Is the `fromHop` convention used consistently in all function signatures?

### 7. Prose-state-machine alignment
Prior bugs were often prose-state-machine mismatches where math was correct but description was wrong. Verify surrounding prose says the same thing as pseudocode — particularly Finalization, Allocation, and Rollover sections.

---

## Suggested pressure-test scenarios

Run through the pseudocode and verify both invariants:

1. Fully subscribed base fund (all hops oversubscribed)
2. Expansion triggered by capped_demand ≥ $1.5M
3. Hop-0 only, $1.0M ≤ capped_demand < $1.5M → refundMode (net_proceeds = $798k < $1M)
4. Hop-0 only, capped_demand = $1.5M → expands and succeeds (net_proceeds = $1,197k)
5. Hop-0 empty, all demand in hop-1 → hop-1 absorbs up to remaining_available
6. Zero hop-2 demand → floor unused, becomes unsold_arm
7. Single address at all three hops → ARM accumulated from all three independently, no refund penalty
8. Full subtree fill: 1 seed fills all slots (1 hop-0 + 3 hop-1 + 6 hop-2 = 10 positions, $33k via one address) → USDC/ARM invariants hold; capped_demand includes all positions
9. 46 seeds each filling full subtrees → capped_demand = 46 × $33k = $1.518M, expansion triggers
10. 67 seeds at hop-0 + hop-1 → capped_demand = $1.273M (base), net_proceeds ≈ $1.066M, succeeds
11. Over-cap same-hop deposit → over-cap portion refunded, allocation at cap
12. Cancel before any commits → withdrawUnallocatedArm() sweeps 1.8M, claimRefund() returns nothing
13. refundMode branch: verify `finalized = true` AND `refundMode = true` both set; verify `cancel()` subsequently reverts
14. Multi-slot: address invited 3× to hop-1 by same seed → cap = 3 × $4k = $12k; 6 outgoing hop-2 slots
15. Multi-slot: address invited by two different seeds to hop-1 → cap = 2 × $4k = $8k; 4 outgoing hop-2 slots
16. Invite link: `commitWithInvite()` with valid signature → creates slot + records commitment atomically; `Invited` and `Committed` events in same tx
17. Invite link: nonce already consumed → revert
18. Invite link: expired deadline → revert
19. Invite link: `revokeInviteNonce(nonce)` then attempt redemption → revert
20. Invite link: `revokeInviteNonce(0)` → revert (nonce 0 reserved)
21. Settlement invariant: for each address, verify `sum(AllocatedHop.armAmount) == Allocated.totalArmAmount`
22. Phased settlement: `emitSettlement()` batches are monotonic; `SettlementComplete` emitted on final batch; `claim()` works before events are fully emitted

---

## Format your findings as

**Must fix** — blocks implementation/audit

**Should fix** — clarity issue an implementer or auditor would need to resolve by inference

**Minor** — wording or cosmetic issues that don't affect correctness

For each: state the issue, location, and the correct fix.
