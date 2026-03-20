Here’s a **v0.1 Implementation Test Spec** you can hand to Ian.

It is written as a contract-facing test plan, not a marketing or review doc. It assumes **`CROWDFUND.md` is canonical**, while the observer and committer specs define event/interface compatibility the implementation must preserve.   

---

# Armada Crowdfund — Implementation Test Spec (v0.1)

## 1. Purpose

Verify that the contract implementation matches the canonical crowdfund mechanism exactly in:

* state transitions
* accounting invariants
* allocation / refund behavior
* invite / slot mechanics
* phased settlement behavior
* emitted event surface required by the observer and committer UIs

This test spec is the bridge between `CROWDFUND.md` and the eventual Foundry test suite. The implementation is not ready for audit until all tests and invariants in this document pass.  

## 2. Sources of Truth

### Canonical contract source

* `CROWDFUND.md` — mechanism, state machine, contract interface, allocation logic, settlement logic, event semantics. 

### Contract-surface compatibility sources

* `CROWDFUND_OBSERVER.md` — event-consumed read model, phased settlement expectations, per-hop settlement display assumptions. 
* `CROWDFUND_COMMITTER.md` — claim timing assumptions, invite-link lifecycle expectations, settlement-data requirements for the connected user. 

### Review / traceability source

* `CROWDFUND_REVIEW_BRIEF.md` — state-machine completeness checklist, pressure-test scenarios, known historical bug classes. 

## 3. Test Harness Assumptions

The test harness should support:

* mocked ERC20 USDC with 6 decimals
* mocked ARM token with 18 decimals
* timestamp control for pre-deadline / post-deadline / expiry / 3-year claim deadline
* role impersonation for:

  * deployer
  * treasury
  * launch team / ROOT authority
  * Security Council
  * arbitrary participants
* toggling between:

  * single-transaction settlement path
  * phased settlement path using `emitSettlement(startIndex, count)` 

The harness should also include helpers for:

* creating seed graphs
* creating direct invites
* creating EIP-712 invite links
* generating EIP-1271-valid signatures
* forcing oversubscription at specific hops
* building repeated same-address and same-hop slot structures

## 4. Global Invariants

These must be tested both in deterministic scenarios and with fuzzing.

### 4.1 USDC conservation

For any finalized success-path outcome:

`net_proceeds + sum(refunds) == sum(total_deposited)`

For refundMode or cancel:

* no treasury transfer occurs
* participants can recover full refundable USDC per spec path. 

### 4.2 ARM conservation

At all times after settlement:

`allocated_arm + unsold_arm == MAX_SALE`

Where `MAX_SALE = 1,800,000 ARM` preloaded. 

### 4.3 Settlement invariant

For each address on the success path:

`sum(AllocatedHop.armAmount) == Allocated.totalArmAmount`

Also:

* `AllocatedHop` is only emitted when `armAmount > 0`
* no `AllocatedHop` exists for zero-allocation `(address, hop)` pairs.  

### 4.4 RefundMode event invariant

If `refundMode == true`:

* `Finalized(refundMode=true)` is emitted
* **no** `Allocated`
* **no** `AllocatedHop`
* `claim()` reverts
* `claimRefund()` returns full deposits.  

### 4.5 Cancel invariant

After `cancel()`:

* `commit()` reverts
* `finalize()` reverts
* `claimRefund()` is available for full deposits
* `withdrawUnallocatedArm()` can sweep all preloaded ARM
* state is irreversible. 

### 4.6 Slot / cap invariant

Participation is slot-based:

* hop-0 slots come from `SeedAdded`
* hop-1 and hop-2 slots come from `Invited`
* each slot increases cap by `HOP_CAP[hop]`
* outgoing invite rights scale with slots held at the inviter’s hop.  

### 4.7 Nonce invariant

* `nonce == 0` is reserved for direct invites only
* `commitWithInvite()` requires `nonce > 0`
* `revokeInviteNonce(0)` reverts
* each nonzero nonce can be consumed or revoked at most once. 

### 4.8 Phased settlement invariant

If phased settlement is used:

* settlement batches are monotonic, sequential, non-overlapping
* no re-emission for already emitted participants
* `SettlementComplete` is emitted exactly once, on the final batch only.  

### 4.9 Commitment amount invariant

* `commit()` requires `amount > 0`
* `commitWithInvite()` requires `amount > 0`
* Zero-amount commits revert — no free slot acquisition without capital.

### 4.10 Seed uniqueness invariant

* `addSeed(address)` reverts if the address is already a seed
* Each address can hold at most one hop-0 slot
* Duplicate seeding is not a path to doubled hop-0 cap or invite rights.

### 4.11 Settlement event ordering invariant

For each participant in a settlement batch (single-tx or phased):

* `Allocated` is emitted before that participant's `AllocatedHop` events
* In phased mode, the same per-participant ordering is preserved within each batch.

### 4.12 Settlement mode equivalence invariant

Single-tx and phased settlement must produce identical economic outputs:

* same `Allocated.totalArmAmount` per address
* same `Allocated.totalRefundAmount` per address
* same `AllocatedHop.armAmount` per (address, hop)
* same unsold ARM outcome
* only event timing differs, not economics.

## 5. Per-Function Test Matrix

Each external function must have:

* happy-path tests
* revert tests
* event tests
* state-change tests
* asset-movement tests

### 5.1 `loadArm()`

Test:

* succeeds exactly once with full preload amount
* reverts if called twice
* reverts if ARM balance/preload is insufficient
* emits `ArmLoaded`
* enables invite/commit window behavior only after successful call. 

### 5.2 `addSeed(seed)`

Test:

* callable only by authorized launch-team/ROOT authority
* only during allowed week-1 window
* only after ARM loaded
* creates hop-0 participation slot
* emits `SeedAdded`
* increments seed count
* enforces seed cap (150)
* **reverts if address is already a seed** — no duplicate hop-0 slots. 

### 5.3 `launchTeamInvite(invitee, fromHop)`

Test:

* callable only by authorized launch-team/ROOT authority
* only during week 1
* only after ARM loaded
* only for `fromHop = 0` (invitee joins hop-1) or `fromHop = 1` (invitee joins hop-2)
* **reverts for `fromHop >= 2`** — no hop-3+ creation
* emits `Invited(..., hop = fromHop + 1, nonce = 0)`
* creates participation slot for invitee
* decrements launch-team budget for the target hop separately (60 hop-1, 60 hop-2)
* verify both budget paths independently exhaust at 60. 

### 5.4 `invite(invitee, fromHop)`

Test:

* requires ARM loaded
* requires pre-deadline
* requires not cancelled
* requires not finalized
* requires inviter has available slots at `fromHop`
* creates invite edge and new participation slot at `fromHop + 1`
* emits `Invited(..., nonce = 0)`
* decrements inviter slot count
* permits duplicate same-hop invites to same address
* duplicate same-hop invites affect graph/slots/cap exactly once per invite. 

### 5.5 `commitWithInvite(inviter, fromHop, nonce, deadline, signature, amount)`

Test:

* signature valid for EOA via EIP-712
* signature valid for contract wallet via EIP-1271
* reverts on expired deadline
* reverts on used nonce
* reverts on revoked nonce
* reverts on `nonce == 0`
* **reverts on `amount == 0`** — no free slot acquisition
* reverts if inviter lacks available slot
* requires ARM loaded, pre-deadline, not cancelled, not finalized
* atomically emits `Invited(..., nonce)` and `Committed(...)`
* decrements inviter slots
* consumes nonce exactly once
* creates slot and cap at `fromHop + 1`
* raw deposit may exceed cap and remains refundable later. 

### 5.6 `revokeInviteNonce(nonce)`

Test:

* reverts on `nonce == 0`
* reverts if already consumed
* reverts if already revoked
* marks nonce revoked permanently
* emits `InviteNonceRevoked`. 

### 5.7 `commit(hop, amount)`

Test:

* requires participation slot at that hop
* hop-0 slot source is `SeedAdded`
* hop-1/hop-2 slot source is `Invited`
* requires ARM loaded, pre-deadline, not cancelled, not finalized
* **requires `amount > 0`** — zero-amount commits revert
* transfers USDC into escrow
* emits `Committed`
* allows over-cap deposits
* raw deposit can exceed effective capped amount without reverting. 

### 5.8 `finalize()`

Test success path:

* only post-deadline
* only if not finalized
* only if not cancelled
* only if `capped_demand >= MINIMUM_RAISE`
* computes allocations and refunds correctly
* sets `finalized = true`
* transfers net proceeds to treasury
* single-tx mode emits `Finalized`, `Allocated`, `AllocatedHop`
* phased mode emits only `Finalized`
* **event ordering:** for each participant, `Allocated` precedes that participant's `AllocatedHop` events in log order. 

Test refundMode branch:

* if post-allocation `net_proceeds < MINIMUM_RAISE`, sets `finalized = true` and `refundMode = true`
* emits `Finalized(refundMode=true)`
* records no allocations
* transfers no treasury proceeds
* emits no settlement events. 

### 5.9 `emitSettlement(startIndex, count)`

Test:

* callable only in phased mode
* only after successful non-refundMode finalization
* rejects overlap / re-emission / out-of-order batches
* emits `Allocated` and `AllocatedHop` for batch
* final batch emits `SettlementComplete` exactly once
* intermediate batches do not. 

### 5.10 `claimRefund()`

Test eligibility paths:

1. `refundMode == true`
2. post-deadline, not finalized, `capped_demand < MINIMUM_RAISE`
3. cancelled

Test:

* refund amount equals full deposit in refundMode/cancel/pre-finalization-min-fail cases
* partial refund path equals recorded refund on success path
* multiple claims revert or zero out correctly
* emits `RefundClaimed`. 

### 5.11 `claim(delegate)`

Test:

* only after successful finalization
* reverts in refundMode
* reverts after 3-year deadline
* requires delegation parameter
* transfers ARM
* records claim state
* emits `ArmClaimed`
* does not affect refund claimability. 

### 5.12 `cancel()`

Test:

* callable only pre-finalization by Security Council
* sets permanent cancelled state
* emits `Cancelled`
* disables new commits and finalization
* enables full refund claims and full ARM sweep. 

### 5.13 `withdrawUnallocatedArm()`

Test windows:

* immediate post-finalization unsold ARM sweep
* post-3-year deadline sweep of remaining ARM balance
* post-cancel sweep of full preloaded ARM

Test that each window only sweeps amounts authorized in that state.

Additional:

* **idempotency:** second call after a successful sweep transfers zero (or reverts) — no double-sweep
* **3-year boundary:** test exact timestamp at claim deadline — verify `claim()` reverts at deadline+1 and sweep becomes available. 

## 6. Deterministic Scenario Tests

These should be named fixtures.

### S1. Fully subscribed base sale

All hops oversubscribed below expansion trigger.
Verify:

* base sale size
* correct hop-0 / hop-1 ceilings
* hop-2 floor + leftover behavior
* pro-rata outputs
* both conservation invariants

### S2. Expansion threshold exact hit

`capped_demand == 1,500,000` exactly.
Verify:

* `sale_size = MAX_SALE`
* no off-by-one at threshold. 

### S3. Hop-0 only, base-size qualifying demand but insufficient net proceeds

Example: raw/capped demand between $1.0M and $1.5M all at hop-0.
Verify:

* finalization allowed to run
* post-allocation `net_proceeds < MINIMUM_RAISE`
* refundMode entered
* full refunds only
* no allocations recorded. 

### S4. Hop-0 only, expansion success

All demand at hop-0, `capped_demand >= 1.5M`.
Verify:

* expansion occurs
* hop-0 ceiling rises to expanded value
* success path can finalize. 

### S5. Hop-0 empty, all demand in hop-1

Verify:

* hop-1 effective ceiling can absorb remaining available
* no over-allocation
* rollover behavior correct. 

### S6. Zero hop-2 demand

Verify:

* floor reserved
* no hop-2 allocation
* reserved capacity contributes to unsold ARM, not phantom USDC. 

### S7. Single address at hops 0, 1, 2

Verify:

* allocations accumulate across all three hops
* no lowest-hop deduplication
* address receives multi-hop settlement
* per-hop settlement sums to aggregate. 

### S8. Recursive self-fill subtree

Seed self-invites and recursively fills downstream slots.
Verify:

* slot counts scale correctly
* caps scale correctly
* graph-legible structure translates to correct economic rights
* expected max-entity exposure is reachable only via the documented slot pattern. 

### S9. Duplicate same-hop invites

Same address receives multiple invites to same hop.
Verify:

* multiple slots created
* cap scales with slot count
* outgoing invite rights scale when appropriate
* duplicate edges do not corrupt settlement or display assumptions. 

### S10. Cancel before finalization

Verify:

* cancel path works from live window
* no commits/finalize afterward
* full refunds available
* all ARM sweepable. 

### S11. Phased settlement

Verify:

* `finalize()` stores settlement but emits only `Finalized`
* batches emit settlement data
* UI-distinguishing signals exist: pre-complete vs complete
* `SettlementComplete` exactly once.  

### S12. No-allocation participant after success

Address commits but receives zero ARM allocation due to full oversubscription at their hop(s).
Verify:

* `Allocated(participant, 0, fullRefundAmount)` is still emitted — every participating address gets an `Allocated` event on the success path, even with zero ARM
* **no** `AllocatedHop` events for that participant (armAmount = 0 at all hops → no per-hop events)
* settlement invariant holds: `sum(AllocatedHop.armAmount) == 0 == Allocated.totalArmAmount`
* full deposit returned via `claimRefund()` (refund = total deposited)
* observer and committer can distinguish "zero allocation with full refund" from "settlement not yet emitted".

### S13. Over-issued invite links

Inviter creates 5 links for 3 available slots. First 3 redeemed succeed.
Verify:

* 4th and 5th redemptions revert with "inviter lacks available slot"
* first 3 slots correctly created
* nonces 4 and 5 remain unconsumed (not consumed-and-failed).

### S14. Zero-amount commit attempts

Test `commit(hop, 0)` and `commitWithInvite(..., amount=0)`.
Verify:

* both revert
* no slot created via zero-amount `commitWithInvite`
* no graph noise from zero-value commits.

### S15. Duplicate seed attempt

`addSeed(alice)` succeeds. `addSeed(alice)` again.
Verify:

* second call reverts
* alice has exactly 1 hop-0 slot, $15k cap, 3 outgoing invites
* seed count incremented only once.

### S16. Maximum network gas fixture

150 seeds, each with full subtrees (1,740 nodes total).
Verify:

* `finalize()` gas usage measured
* determines whether phased settlement is needed
* both invariants hold at scale.

### S17. Settlement mode equivalence

Run the same commitment set through single-tx and phased settlement.
Verify:

* identical `Allocated.totalArmAmount` per address
* identical `Allocated.totalRefundAmount` per address  
* identical `AllocatedHop.armAmount` per (address, hop)
* identical unsold ARM outcome
* only event timing differs.

### S18. Settlement event ordering

Finalize with multiple participants at multiple hops.
Verify:

* for each participant, `Allocated` log index < all of that participant's `AllocatedHop` log indices
* in phased mode, same ordering within each batch.

### S19. Timestamp boundaries

Test exact-second behavior at:

* week-1 invite window close (day 7 → day 8)
* commitment deadline (day 21)
* invite link expiry (deadline field)
* 3-year claim deadline

Verify correct revert/accept at boundary and boundary ± 1 second.

## 7. Property / Fuzz Tests

Run fuzzing over:

* number of seeds
* invite graph shape
* duplicate invite patterns
* same-address multi-hop occupation
* same-address same-hop slot multiplication
* over-cap deposits
* demand distributions by hop
* phased batch sizes
* claim order
* refund claim order
* nonce creation / revocation / redemption order
* **timestamp proximity to boundaries** (week-1 close, commitment deadline, link expiry, 3-year claim deadline)
* **settlement mode** (same commitment set run through single-tx and phased — assert identical economics)

Fuzz invariants:

* conservation laws
* settlement invariant
* settlement mode equivalence (same economic outputs regardless of single-tx vs phased)
* no negative refunds / allocations
* no over-consumption of slots
* no duplicate nonce consumption
* no successful invite redemption after revoke or expiry
* no event emission forbidden by state
* no zero-amount commit succeeds
* no duplicate seed succeeds

## 8. Event Correctness Tests

Verify exact emitter / fields / absence conditions for:

* `ArmLoaded`
* `SeedAdded`
* `Invited`
* `Committed`
* `InviteNonceRevoked`
* `Finalized`
* `Allocated`
* `AllocatedHop`
* `SettlementComplete`
* `ArmClaimed`
* `RefundClaimed`
* `Cancelled`  

Specific checks:

* direct invites emit `Invited(..., nonce = 0)`
* link invites emit `Invited(..., nonce > 0)`
* `AllocatedHop` emitted only when `armAmount > 0`
* no settlement events in refundMode
* `SettlementComplete` only in phased mode
* **settlement event ordering:** for each participant, `Allocated` log index < all of that participant's `AllocatedHop` log indices within the same transaction or batch
* `commitWithInvite()` emits `Invited` and `Committed` in the same transaction — verify both present with correct fields

## 9. UI-Contract Compatibility Tests

These are not browser tests. They are contract-surface tests to protect the UI assumptions.

### Observer compatibility

Contract events must be sufficient for the observer to reconstruct:

* slot-based node graph from `SeedAdded` + `Invited`
* per-hop commitments from `Committed`
* aggregate settlement from `Allocated`
* per-hop settlement from `AllocatedHop`
* phased completeness from `SettlementComplete`
* refundMode and cancel states from `Finalized` / `Cancelled`. 

### Committer compatibility

Contract events and state must be sufficient for the committer to determine:

* connected address eligibility from `SeedAdded` and `Invited`
* link lifecycle from `Invited.nonce` + `InviteNonceRevoked`
* claim tab states:

  * `Allocated` present
  * `SettlementComplete` present without `Allocated`
  * neither yet
* refundMode behavior with no settlement events. 

## 10. Exit Criteria

Implementation is ready for audit only when:

* all per-function happy-path tests pass
* all revert tests pass
* all deterministic scenarios pass (S1–S19)
* all global invariants pass under fuzzing (4.1–4.12)
* settlement mode equivalence confirmed (single-tx and phased produce identical economics)
* all event-surface tests pass including ordering guarantees
* observer and committer compatibility tests pass
* every major mechanism rule in `CROWDFUND.md` maps to at least one explicit test in `SPEC_TRACEABILITY.md`

---

## What should be produced from this

* `Crowdfund.t.sol` — per-function unit tests (§5)
* `CrowdfundScenarios.t.sol` — deterministic scenario fixtures (§6, S1–S19)
* `CrowdfundInvariants.t.sol` — invariant / fuzz harness (§4, §7)
* `CrowdfundEvents.t.sol` — event correctness and ordering tests (§8)
* `CrowdfundUiSurface.t.sol` — observer / committer compatibility tests (§9)
* `CrowdfundGas.t.sol` — gas measurement fixtures (S16: max-network finalization)
* `SPEC_TRACEABILITY.md` — mapping from spec rules to tests; every rule in `CROWDFUND.md` must appear

