# Armada Crowdfund — Monitoring & Alerting Spec

## 1. Purpose

Define the canonical signals, derived metrics, alert thresholds, and operator-response mappings for the Armada crowdfund.

This document is intentionally **tooling-agnostic**. It specifies **what must be monitored** and **when an alert should fire**, but does not require any particular stack (indexer, webhook, bot, dashboard vendor, pager, etc.). Alert delivery channels are placeholders until infrastructure is chosen.

This spec is complementary to:

- `CROWDFUND.md` — canonical contract and event behavior
- `OPERATIONS.md` — operator procedures and failure handling
- `CROWDFUND_OBSERVER.md` — event-consumed read model
- `CROWDFUND_COMMITTER.md` — invite and claim UX assumptions

---

## 2. Scope

**In scope:**
- Canonical on-chain signals to monitor
- Derived operational state
- Threshold-based alerts
- Alert severity classes
- Runbook mapping to `OPERATIONS.md`
- Monitoring requirements across all lifecycle phases

**Out of scope:**
- Alert transport implementation
- Dashboard vendor choice
- Paging / on-call rotations
- Participant-facing communications templates

---

## 3. Monitoring Principles

1. **Event-first.** State must be derivable from contract events, matching the observer model. Monitoring must not depend on frontend-local state unavailable to operators.
2. **Thresholds must map to action.** Alerts without a corresponding procedure in `OPERATIONS.md` should not exist.
3. **Distinguish normal-but-sensitive from abnormal.** `refundMode` is a defined crowdfund outcome, not a security exploit. Duplicate same-hop invites are an explicit design feature, not misuse. Alert text must reflect this.
4. **Derived metrics must match the canonical mechanism.** In particular: slot-based participation (`participation_slots × HOP_CAP[hop]`), duplicate same-hop invites, phased settlement, and the `AllocatedHop` / `Allocated` settlement model must be treated as intentional.
5. **No monitoring state is authoritative over contract state.** When monitoring and contract state diverge, contract state wins. RPC/indexer issues are a monitoring failure, not a contract failure.

---

## 4. Severity Levels

| Level | Name | Response time | Examples |
|---|---|---|---|
| P0 | **Immediate** | < 1 hour | `Cancelled`; settlement stalled; proceeds mismatch; unexpected settlement events after refundMode |
| P1 | **Same day** | < 8 hours | `refundMode` triggered; contract armed but not open past expected time; deadline passed without finalization |
| P2 | **Attention required** | Next working window | Unusual duplicate-slot growth; seed/budget nearing exhaustion; demand thresholds; claim lag |
| P3 | **Informational** | No action required | `ArmLoaded`; first `SeedAdded`; `SettlementComplete`; successful finalization |

---

## 5. Canonical Event Surface

Monitoring must consume the following events. No monitoring logic may assume state that isn't derivable from this set plus explicit contract state reads.

| Event | Key fields |
|---|---|
| `ArmLoaded` | — |
| `SeedAdded` | `address seed` |
| `Invited` | `address inviter, address invitee, uint8 hop, uint256 nonce` |
| `Committed` | `address participant, uint8 hop, uint256 amount` |
| `InviteNonceRevoked` | `address inviter, uint256 nonce` |
| `Finalized` | `uint256 saleSize, uint256 allocatedArm, uint256 netProceeds, bool refundMode` |
| `Allocated` | `address indexed participant, uint256 totalArmAmount, uint256 totalRefundAmount` |
| `AllocatedHop` | `address indexed participant, uint8 indexed hop, uint256 armAmount` |
| `SettlementComplete` | — |
| `ArmClaimed` | `address participant, uint256 armAmount, address delegate` |
| `RefundClaimed` | `address participant, uint256 usdcAmount` |
| `Cancelled` | — |

Any monitoring that requires additional contract state reads (e.g. `finalized`, `refundMode`, `cancelled` flags; balance reads for treasury verification) must document those reads explicitly.

---

## 6. Derived State Model

Monitoring derives the current lifecycle phase from events and timestamps.

| Phase | Conditions |
|---|---|
| **PRE-ARMED** | No `ArmLoaded` emitted |
| **ARMED / PRE-OPEN** | `ArmLoaded` emitted; `now < openTimestamp` |
| **OPEN / WEEK 1** | `ArmLoaded`; `openTimestamp ≤ now ≤ week1Deadline` |
| **OPEN / WEEKS 2–3** | `ArmLoaded`; `week1Deadline < now ≤ commitmentDeadline` |
| **DEADLINE PASSED / QUALIFIED / NOT FINALIZED** | `now > commitmentDeadline`; derived `capped_demand ≥ MINIMUM_RAISE`; no `Finalized`; no `Cancelled` |
| **DEADLINE PASSED / SUB-MINIMUM / REFUNDS AVAILABLE** | `now > commitmentDeadline`; derived `capped_demand < MINIMUM_RAISE`; no `Finalized`; no `Cancelled`. Refunds auto-available — do NOT call `finalize()`. |
| **FINALIZED / SUCCESS / SINGLE-TX** | `Finalized(refundMode=false)`; `Allocated` + `AllocatedHop` in the **same transaction** as `Finalized`. `SettlementComplete` is **never** emitted in this mode — do not wait for it. |
| **FINALIZED / SUCCESS / PHASED IN PROGRESS** | `Finalized(refundMode=false)`; `SettlementComplete` not yet seen |
| **FINALIZED / SUCCESS / SETTLEMENT COMPLETE** | `SettlementComplete` seen |
| **FINALIZED / REFUND MODE** | `Finalized(refundMode=true)` |
| **CANCELLED** | `Cancelled` emitted |

Phase transitions are monotonic and irreversible (except PRE-ARMED → ARMED → OPEN which follow timestamps).

---

## 7. Derived Metrics

### 7.1 Slot-based participation

Monitoring must use the **`participation_slots` model**, not naive unique-address counts.

- **hop-0 slots:** count of `SeedAdded` events (always 0 or 1 per address)
- **hop-1 / hop-2 slots:** count of `Invited` events where `invitee = address` and `hop = target_hop`
- **Per-address cap at hop:** `participation_slots[(address, hop)] × HOP_CAP[hop]`
- **Inviter slots remaining:** `participation_slots[(inviter, fromHop)] × HOP_INVITE_LIMIT[fromHop] − invites_sent_from_(inviter, fromHop)`

  Where `HOP_INVITE_LIMIT` is (from `CROWDFUND.md` Hop Structure table):
  - `HOP_INVITE_LIMIT[0] = 3` (hop-0 nodes may invite up to 3 addresses to hop-1)
  - `HOP_INVITE_LIMIT[1] = 2` (hop-1 nodes may invite up to 2 addresses to hop-2)
  - `HOP_INVITE_LIMIT[2] = 0` (hop-2 nodes may not invite)

  An address with multiple participation slots at a hop has proportionally more outgoing invite capacity. Using the flat `HOP_INVITE_LIMIT[fromHop] − invites_sent` formula will undercount remaining capacity for any address with duplicate slots at that hop.

Monitoring dashboards must never present raw `Committed` amounts as if they were capped demand. `capped_demand` is the sum of `min(total_committed_at_hop, participation_slots × HOP_CAP[hop])` across all `(address, hop)` pairs.

### 7.2 Commitment metrics

Derive and track separately:

- **Raw deposited USDC** — sum of `Committed.amount` per `(address, hop)`
- **Effective capped demand** — slot-capped sum per `(address, hop)`, aggregated
- **Per-hop demand** — effective capped demand by hop
- **Aggregate `capped_demand`** — sum across all hops and addresses (canonical expansion/minimum-raise variable)

### 7.3 Duplicate same-hop slot count

Track: number of `(address, hop)` pairs where `participation_slots[(address, hop)] > 1`.

This is **intentional design behavior** under the `participation_slots[(address, hop)] × HOP_CAP[hop]` model — each invite creates a new slot. This metric is for operator awareness, not enforcement. See §9.1.

### 7.4 Settlement metrics

Post-finalization, track:

- Count of `Allocated` events emitted vs expected participant count (= count of unique addresses with at least one `Committed` event — includes zero-ARM addresses on the success path)
- Count of `AllocatedHop` events emitted
- Settlement invariant check for sampled addresses: `sum(AllocatedHop.armAmount) == Allocated.totalArmAmount`
- `SettlementComplete` seen / not seen (phased mode only)
- Time elapsed since `Finalized` without `SettlementComplete` (phased mode)

### 7.5 Claims metrics

- ARM claimed: `ArmClaimed` count and total ARM vs `Finalized.allocatedArm`
- Refunds claimed: `RefundClaimed` count and total USDC vs expected total refundable
- Participation rates as percentages over time

---

## 8. Alert Rules

### A1 — ARM loaded

| Field | Value |
|---|---|
| **Signal** | `ArmLoaded` emitted |
| **Severity** | P3 |
| **Meaning** | Sale is armed. Commitment window opens at `openTimestamp`. |
| **Runbook** | `OPERATIONS.md` §3 Steps 5–8 |

---

### A2 — Sale should be open but not yet armed

| Field | Value |
|---|---|
| **Signal** | No `ArmLoaded` |
| **Condition** | `now ≥ openTimestamp` and no `ArmLoaded` |
| **Severity** | P1 |
| **Meaning** | Launch sequence incomplete; commitments cannot begin |
| **Runbook** | `OPERATIONS.md` §3 Steps 4–5 |

---

### A3 — Week-1 action outside week-1 window

| Field | Value |
|---|---|
| **Signal** | `SeedAdded` or ROOT-issued `Invited` |
| **Condition** | Event timestamp after `week1Deadline` |
| **Severity** | P0 |
| **Meaning** | Contract or monitoring assumptions broken — this should not be possible |
| **Runbook** | `OPERATIONS.md` §9 failure investigation; Security Council review |

---

### A4 — Seed budget thresholds

| Field | Value |
|---|---|
| **Signal** | Derived seed count from `SeedAdded` |
| **Condition** | Seed count reaches 80%, 90%, 100% of configured budget (150) |
| **Severity** | P2 at 80%/90%; P1 at 100% |
| **Meaning** | Week-1 hop-0 expansion capacity running low |
| **Runbook** | `OPERATIONS.md` §4 Week-1 go/no-go checkpoint; §10 decision log |

---

### A5 — Launch-team placement budget thresholds

| Field | Value |
|---|---|
| **Signal** | COUNT of ROOT-issued `Invited` events, by `hop` field (where `hop` in the `Invited` event is the invitee's hop level — not the `fromHop` parameter used when calling `launchTeamInvite`) |
| **Condition** | Hop-1 or hop-2 placement count reaches 80%, 90%, 100% of budget (60 each) |
| **Severity** | P2 at 80%/90%; P1 at 100% |
| **Meaning** | Week-1 discretionary placement capacity running low |
| **Runbook** | `OPERATIONS.md` §4 Week-1 operations; §10 decision log |

---

### A6 — Duplicate same-hop slot growth

| Field | Value |
|---|---|
| **Signal** | Derived count of `(address, hop)` pairs where `participation_slots > 1` |
| **Condition** | Count exceeds configured watch threshold, or grows materially faster than expected baseline |
| **Severity** | P2 |
| **Meaning** | Expected under the design — each invite creates a new slot. Alert is for awareness only. |
| **Runbook** | `OPERATIONS.md` §4/§5 monitoring; no automatic intervention |
| **Note** | See §9.1. Do not treat as exploit. |

---

### A7 — Expansion threshold approaching

| Field | Value |
|---|---|
| **Signal** | Derived `capped_demand` |
| **Condition** | Reaches 80%, 90%, 95%, 100% of `EXPANSION_TRIGGER` ($1,500,000) |
| **Severity** | P2 |
| **Meaning** | Sale may expand to MAX_SALE (1.8M ARM); allocation model shifts |
| **Runbook** | `OPERATIONS.md` §5 pre-finalization checkpoint |

---

### A8 — Minimum raise at risk late in sale

| Field | Value |
|---|---|
| **Signal** | Derived `capped_demand` vs `MINIMUM_RAISE` ($1,000,000) |
| **Condition** | `capped_demand < MINIMUM_RAISE` with <72h remaining; then <24h remaining |
| **Severity** | P2 |
| **Meaning** | RefundMode risk increasing. Note: demand often concentrates near deadline. |
| **Runbook** | `OPERATIONS.md` §5 Weeks 2–3 cadence; §11 Checkpoint 3 |

---

### A9a — Deadline passed, finalization needed

| Field | Value |
|---|---|
| **Signal** | Absence of `Finalized` and `Cancelled` |
| **Condition** | `now > commitmentDeadline` AND derived `capped_demand ≥ MINIMUM_RAISE` |
| **Severity** | P1 initially; P0 if unresolved beyond configured grace window (e.g. 2 hours) |
| **Meaning** | Sale qualified — finalization action required. Someone must call `finalize()`. |
| **Runbook** | `OPERATIONS.md` §11 Checkpoint 3; §6 Finalization procedure |

---

### A9b — Deadline passed, refunds auto-available

| Field | Value |
|---|---|
| **Signal** | Absence of `Finalized` and `Cancelled` |
| **Condition** | `now > commitmentDeadline` AND derived `capped_demand < MINIMUM_RAISE` |
| **Severity** | P1 |
| **Meaning** | Sale did not qualify. `claimRefund()` eligibility activates automatically — do **not** call `finalize()`. Announce to participants immediately. |
| **Runbook** | `OPERATIONS.md` §5 pre-finalization checkpoint (capped_demand < MINIMUM_RAISE branch) |

---

### A10 — RefundMode triggered

| Field | Value |
|---|---|
| **Signal** | `Finalized(refundMode=true)` |
| **Severity** | P1 |
| **Meaning** | Sale did not reach minimum net proceeds after allocation. Participants can claim full refunds. **Not an exploit.** |
| **Runbook** | `OPERATIONS.md` §6 Path C (refundMode); §9.7 |

---

### A11 — Cancel triggered

| Field | Value |
|---|---|
| **Signal** | `Cancelled` |
| **Severity** | P0 |
| **Meaning** | Crowdfund permanently cancelled by Security Council |
| **Runbook** | `OPERATIONS.md` §7 cancel procedure |

---

### A12 — Successful finalization

| Field | Value |
|---|---|
| **Signal** | `Finalized(refundMode=false)` |
| **Severity** | P3 |
| **Meaning** | Sale settled successfully |
| **Runbook** | `OPERATIONS.md` §6 post-finalization verification; §8 |

---

### A13 — Phased settlement started

| Field | Value |
|---|---|
| **Signal** | `Finalized(refundMode=false)` without `Allocated` events in the same transaction |
| **Severity** | P2 |
| **Meaning** | Settlement is valid but event emission is incomplete. Operators must call `emitSettlement()`. |
| **Runbook** | `OPERATIONS.md` §6 Path B (phased finalization) |

---

### A14 — Phased settlement stalled

| Field | Value |
|---|---|
| **Signal** | `Finalized(refundMode=false)` seen; `SettlementComplete` not yet seen |
| **Condition** | Time elapsed since `Finalized` exceeds configured threshold (e.g. 4 hours) |
| **Severity** | P0 |
| **Meaning** | Settlement event emission is incomplete too long after finalization. Participants cannot see allocations. |
| **Runbook** | `OPERATIONS.md` §6 Path B; §9.6 |

---

### A15 — Settlement complete

| Field | Value |
|---|---|
| **Signal** | `SettlementComplete` |
| **Severity** | P3 |
| **Meaning** | All `Allocated` and `AllocatedHop` events emitted in phased mode |
| **Runbook** | `OPERATIONS.md` §6 Path B Step 3 |

---

### A16 — Treasury proceeds mismatch

| Field | Value |
|---|---|
| **Signal** | `Finalized(refundMode=false)` plus `USDC.balanceOf(treasury)` read |
| **Condition** | Treasury USDC balance increase ≠ `Finalized.netProceeds` |
| **Severity** | P0 |
| **Meaning** | Accounting mismatch or integration failure |
| **Runbook** | `OPERATIONS.md` §8 proceeds verification |

---

### A17 — Unexpected settlement events after refundMode or cancel

| Field | Value |
|---|---|
| **Signal** | `Allocated`, `AllocatedHop`, or `SettlementComplete` |
| **Condition** | Emitted after `Finalized(refundMode=true)` or after `Cancelled` |
| **Severity** | P0 |
| **Meaning** | Critical contract or event-surface violation. Should never occur. |
| **Runbook** | Immediate investigation; treat as severe implementation bug |

---

### A18 — ARM claims participation lag

| Field | Value |
|---|---|
| **Signal** | `ArmClaimed` count vs allocated participant count |
| **Condition** | <50% of allocated participants claimed after 14 days post-finalization |
| **Severity** | P2 |
| **Meaning** | Participant awareness issue; not a contract failure |
| **Runbook** | `OPERATIONS.md` §8 claims monitoring |

---

### A19 — Refund participation lag

| Field | Value |
|---|---|
| **Signal** | `RefundClaimed` total vs total refundable USDC |
| **Condition** | >10% of refundable USDC unclaimed after 30 days |
| **Severity** | P2 |
| **Meaning** | Participants may need reminders |
| **Runbook** | `OPERATIONS.md` §8 claims monitoring |

---

### A20 — 3-year sweep window reached

| Field | Value |
|---|---|
| **Signal** | Elapsed time since finalization |
| **Condition** | `now > finalization_timestamp + (3 × 365 × 24 × 3600)` — strict post-deadline, matching `OPERATIONS.md` §8 and the contract's sweep eligibility: claims are available *through* the deadline; sweep is available only *after*. |
| **Severity** | P2 |
| **Meaning** | Unclaimed ARM is now sweepable via `withdrawUnallocatedArm()` |
| **Runbook** | `OPERATIONS.md` §8 3-year deadline sweep |

---

## 9. Special Monitoring Notes

### 9.1 Duplicate same-hop invites are not an exploit

Under the `participation_slots[(address, hop)] × HOP_CAP[hop]` model, each invite to a given hop from any inviter creates a new participation slot. Multiple inviters — or the same inviter issuing multiple invites — each increase the invitee's effective cap at that hop. This is specified behavior, not misuse.

Alert A6 exists for operator **awareness**, not for automatic escalation or remediation. The appropriate question when duplicate slots appear is: "is this consistent with expected seed behavior?" not "is this an attack?"

### 9.2 `refundMode` is not a security incident

`refundMode` is a defined crowdfund outcome that occurs when `capped_demand ≥ MINIMUM_RAISE` but `net_proceeds < MINIMUM_RAISE` after allocation — typically at base size when hop-0 is oversubscribed and later-hop demand doesn't close the gap. It cannot occur after expansion.

Alert A10 is P1 (not P0) because operators must shift participant guidance immediately, but there is no security threat. Alert text and participant communications must avoid exploit-like framing.

### 9.3 Phased settlement is valid

Absence of `Allocated` / `AllocatedHop` events immediately after `Finalized(refundMode=false)` is not automatically a bug. The correct sequence is:

1. Alert A12 fires (successful finalization — P3)
2. If `Allocated` events are absent from the same transaction: Alert A13 fires (phased settlement started — P2)
3. Operators begin calling `emitSettlement()` batches
4. If `SettlementComplete` doesn't arrive within threshold: Alert A14 fires (stalled — P0)
5. On `SettlementComplete`: Alert A15 fires (complete — P3)

### 9.4 Zero-allocation addresses on success path

A participant may receive an `Allocated` event with `totalArmAmount = 0` and a non-zero `totalRefundAmount`, while emitting no `AllocatedHop` event. This is valid — the address committed but received no ARM due to oversubscription. Monitoring must treat this as correct.

### 9.5 `capped_demand` calculation

Monitoring must calculate `capped_demand` using the slot-based cap per `CROWDFUND.md`: for each `(address, hop)`, cap = `participation_slots[(address, hop)] × HOP_CAP[hop]`. Aggregating raw `Committed` amounts without applying slot-based caps will overstate demand and misfire threshold alerts.

---

## 10. Required Dashboard Views

Operators must have read access to the following views before and throughout the commitment window.

### 10.1 Lifecycle view

- Current derived phase
- `openTimestamp`, `week1Deadline`, `commitmentDeadline`
- Finalization timestamp (if any)
- `refundMode` / `cancelled` flags
- Phased settlement progress / complete status

### 10.2 Budget view

- Seed count used / 150 total
- Launch-team hop-1 placements used / 60 total
- Launch-team hop-2 placements used / 60 total
- Inviter slot consumption per active seed node (where reconstructable from events)

### 10.3 Demand view

- Raw committed USDC (per hop and total)
- Derived `capped_demand` (slot-capped, per hop and total)
- % of `MINIMUM_RAISE` ($1M)
- % of `EXPANSION_TRIGGER` ($1.5M)

### 10.4 Settlement view

- Finalized: success / refundMode / cancelled
- `Allocated` event count vs expected participant count
- `AllocatedHop` event count
- Settlement batch progress (phased mode: % complete, time since `Finalized`)
- `SettlementComplete` seen / not seen

### 10.5 Claims view

- ARM claimed: count and % of allocated participants
- Refund claimed: USDC amount and % of refundable total
- Unclaimed ARM remaining (post-finalization sweep eligibility)

### 10.6 Graph health view

- Occupied `(address, hop)` node count by hop
- Duplicate same-hop slot count (A6 input)
- Same-address multi-hop occupancy (addresses at 2+ hops)
- ROOT-issued placement count by hop

---

## 11. Alert Payload Requirements

Each alert must include:

- Alert ID (e.g. A9)
- Severity (P0–P3)
- Chain ID and contract address
- Current derived phase
- Triggering event(s) or state condition
- Relevant counts / threshold values
- Direct reference to `OPERATIONS.md` section
- Alert destination: `[TBD: alerting channel]`

---

## 12. Runbook Mapping Table

| Alert(s) | `OPERATIONS.md` section |
|---|---|
| A1, A2 | §3 Deployment sequence (Steps 4–8) |
| A3 | §9 Failure scenarios — immediate investigation |
| A4, A5 | §4 Week-1 cadence; §10 Decision log; §11 Checkpoint 2 |
| A6 | §4/§5 Monitoring; no automatic action |
| A7, A8 | §5 Weeks 2–3 cadence; §11 Checkpoint 3 |
| A9a | `OPERATIONS.md` §11 Checkpoint 3; §6 Finalization procedure |
| A9b | `OPERATIONS.md` §5 Weeks 2–3 cadence (sub-minimum branch) |
| A10 | §6 Path C (refundMode); §9.7 |
| A11 | §7 Cancel procedure |
| A12 | §6 post-finalization verification; §8 |
| A13, A14, A15 | §6 Path B (phased finalization); §9.6 |
| A16 | §8 Proceeds verification |
| A17 | Immediate investigation — implementation bug |
| A18, A19 | §8 Claims monitoring |
| A20 | §8 3-year deadline sweep |

---

## 13. Threshold Placeholders

The following thresholds are marked `[TBD]` and must be set before monitoring is deployed. They depend on final infrastructure choices and operational context.

| Alert | Threshold | Default suggestion |
|---|---|---|
| A6 | Duplicate-slot watch threshold | `[TBD]` — start at 10% of occupied hop-1/2 nodes |
| A9a | Grace window after deadline before P0 escalation | `[TBD]` — 2 hours suggested |
| A14 | Stall threshold for phased settlement | `[TBD]` — 4 hours suggested |
| A18 | ARM claim participation floor | 50% of allocated participants after 14 days |
| A19 | Refund claim lag threshold | >10% unclaimed after 30 days |

---

## 14. Exit Criteria

This monitoring spec is implementation-ready when:

- Every alert rule maps to a real signal from the canonical event surface or an explicit contract state read
- Every P0/P1 alert maps to a concrete section of `OPERATIONS.md`
- Phased settlement is handled without ambiguity (A13/A14/A15 sequence)
- Duplicate same-hop slot growth is explicitly treated as valid-but-watchworthy (A6)
- `refundMode`, cancel, and 3-year sweep windows are all covered
- No alert assumes frontend-local state unavailable to operators
- All `[TBD]` thresholds in §13 have been resolved
- `OPERATIONS.md` section references are verified against final headings
