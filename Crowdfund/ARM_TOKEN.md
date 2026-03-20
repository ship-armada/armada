# ARM Token — Contract Behavior Spec

## 1. Purpose and Scope

ARM is the governance and ownership token of the Armada protocol. This document specifies the ARM token contract's behavior — what the crowdfund, governance, and audit layers are allowed to assume about it.

**In scope:** ERC-20 behavior, transfer restrictions, voting/delegation mechanics, minting/burning, admin powers, upgradeability, event surface, and integration assumptions for dependent contracts.

**Out of scope:** Crowdfund mechanics (see CROWDFUND.md), governance proposal lifecycle (see GOVERNANCE.md), tokenomics narrative, market positioning.

**This is a contract behavior spec, not a tokenomics document.**

---

## 2. Canonical Token Properties

| Property | Value |
|---|---|
| Name | `Armada` |
| Symbol | `ARM` |
| Decimals | 18 |
| Total supply | 12,000,000 ARM (12,000,000 × 10^18 smallest units) |
| Supply model | Fixed. All 12,000,000 ARM exist at genesis. No minting after deployment. |
| Burn | None. No burn function. Supply is permanently fixed. |

---

## 3. Genesis Allocation

All ARM is minted directly to final recipients in the constructor. There is no deployer holding — the constructor calls `_mint()` for each recipient, so ARM never passes through an intermediate address.

**Constructor mints:**

| Allocation | Amount | Recipient | Notes |
|---|---|---|---|
| Crowdfund contract | 1,800,000 (MAX_SALE) | Crowdfund contract address | Always 1.8M regardless of elastic expansion. Verified by `loadArm()`. |
| Treasury | 7,800,000 | Treasury multisig | |
| Team + Airdrop | 2,400,000 | Shared revenue-lock contract | See below. |

The constructor also sets name, symbol, and stores the immutable configuration: whitelist addresses (crowdfund, treasury, revenue-lock), treasury address (for delegation revert), `delegateOnBehalf` caller addresses (crowdfund, revenue-lock), and `setTransferable` caller addresses (governor executor, wind-down contract).

**No deployer in the whitelist.** The deployer address never holds ARM and is not whitelisted. This eliminates a trust-surface issue: if the deployer were permanently whitelisted (the whitelist is add-only), it could become a special pre-unlock transferable holder if it ever received ARM while transfers were restricted.

**Revenue-lock contract architecture:** A single shared revenue-lock contract holds all team and airdrop ARM (2,400,000 total). The contract tracks per-beneficiary allocations internally:

- **Beneficiary list** is set at deployment: each team member, each airdrop recipient, and the Knowable Safe (which holds the portion reserved for future contributors) are all entries in the same list with their respective amounts.
- **Release logic** is identical for all beneficiaries: as revenue milestones are reached, each beneficiary can call `release(delegatee)` to withdraw their unlocked percentage. ARM is transferred and delegated atomically.
- **The Knowable Safe** appears in the beneficiary list like any other team member — it simply has a larger allocation. Future contributor allocations are handled through off-chain agreements between Knowable and contributors; once ARM is released to the Safe's wallet (per the milestone schedule), Knowable distributes to contributors through standard ARM transfers (available once governance enables global transfers).
- **One whitelist entry** in the ARM token constructor for this lock contract. No per-recipient whitelisting needed.

**Post-finalization treasury growth:** The crowdfund contract always receives 1,800,000 ARM at deployment. If the sale stays at base size (1,200,000 ARM allocated), the remaining 600,000 ARM becomes unsold and sweepable to treasury via `withdrawUnallocatedArm()` after finalization. The treasury's total ARM holding post-finalization is therefore 7,800,000 + unsold amount.

---

## 4. ERC-20 Behavior

ARM implements standard ERC-20 with voting extensions:

| Function | Behavior |
|---|---|
| `transfer(to, amount)` | Standard. Subject to transfer restriction (§5). |
| `transferFrom(from, to, amount)` | Standard. Subject to transfer restriction (§5). |
| `approve(spender, amount)` | Standard. |
| `allowance(owner, spender)` | Standard. |
| `balanceOf(address)` | Standard. Returns full balance regardless of lock status. |
| `totalSupply()` | Always returns 12,000,000 × 10^18. |

**Extensions:**
- EIP-2612 `permit()` — gasless approvals via signed messages
- Compound-style `Checkpoint` voting (see §6)

**Absent by design:**
- No `blacklist()`, `freeze()`, `pause()`, or `blocklist()` function
- No address-level transfer blocking beyond the global transfer restriction
- No clawback or recovery mechanism
- No fee-on-transfer
- No rebasing

---

## 5. Transfer Restriction State Machine

ARM has a global transfer restriction at launch. This is the most important behavioral property for dependent contracts.

### States

```
                    governance proposal passes
    RESTRICTED ──────────────────────────────────► UNRESTRICTED
    (launch)                                        (permanent)
```

| State | Transfer behavior | Who can transfer | Irreversible? |
|---|---|---|---|
| **RESTRICTED** | `transfer()` and `transferFrom()` revert for most callers | Only whitelisted addresses (see below) | No — transitions to UNRESTRICTED |
| **UNRESTRICTED** | Standard ERC-20 transfers for all addresses | Everyone | Yes — cannot return to RESTRICTED |

### Whitelisted addresses in RESTRICTED state

These addresses can send ARM even while transfers are globally restricted. **The initial whitelist is set in the constructor. Governance can add new addresses via extended proposal, but can never remove an existing whitelisted address.** Once whitelisted, always whitelisted.

**Initial whitelist (constructor-set):**

| Address | Why | Set how |
|---|---|---|
| Crowdfund contract | Must distribute ARM to claimants via `claim()` and `delegateOnBehalf()` | Constructor parameter |
| Treasury | Must send ARM via governance proposals while restricted | Constructor parameter |
| Revenue-lock contract | Must release team/airdrop ARM to beneficiaries as milestones are reached | Constructor parameter |

**Post-deployment additions (governance-gated):**

`addToWhitelist(address)` — callable only by governance (via extended proposal through timelock). Adds an address to the whitelist permanently. There is no `removeFromWhitelist`. This enables:
- Recovery from a wrong address at deployment without redeploying the token
- Future infrastructure contracts that need to transfer ARM while restricted

**Trust surface acknowledgment:** In RESTRICTED state, whitelisted addresses can send ARM while everyone else cannot. Governance adding a new whitelisted address creates a partial transfer exception — that address can send ARM before global unlock. This is an intentional governance power gated by extended proposal (14-day vote, 30% quorum, 7-day execution delay, subject to SC veto). The spec does not constrain whitelist additions to contracts or infrastructure only — governance bears responsibility for evaluating the implications of each addition.

**Why add-only:** The trust-critical property is that whitelisted contracts can never lose their ability to function. Crowdfund participants need to know `claim()` will always work. Team members need to know `release()` will always work. Add-only guarantees this — no governance vote can break an existing whitelisted contract.

**Transfer restriction logic:** A transfer succeeds in RESTRICTED state if and only if the **sender** is a whitelisted address. Non-whitelisted holders cannot send ARM to anyone (including whitelisted addresses) until global transfer unlock. This means:
- The crowdfund contract can send ARM to participants (sender = crowdfund, whitelisted)
- Participants cannot send ARM anywhere (sender = participant, not whitelisted)
- The treasury can send ARM via governance proposals (sender = treasury, whitelisted)
- Revenue-lock contracts can release unlocked ARM to recipients (sender = lock contract, whitelisted)

**Not whitelisted (by design):**

| Address | Why not |
|---|---|
| Governance contract(s) | Proposal bonds (per GOVERNANCE.md: 1,000 ARM, returned after lock period) require a holder to transfer ARM to the governance contract. During RESTRICTED state, non-whitelisted holders cannot transfer — so bonds are technically impossible. But bonds are also economically meaningless before transfer unlock: "losing access" to non-transferable ARM has zero opportunity cost. Bonds activate naturally once governance enables transfers, at which point all addresses can transfer and no whitelist is needed. **Pre-transfer-unlock governance operates on proposal threshold only (12,000 delegated ARM) — no bond required.** This avoids a chicken-and-egg problem: the proposal to enable transfers must itself be creatable without a bond. The governor contract's authority to call `setTransferable(true)` is a function access-control check, not a transfer-whitelist issue — see §8 transfer gate controller role. |
| Knowable Safe | The Safe is a beneficiary in the revenue-lock contract — same as any other team member, just with a larger allocation. ARM is released to the Safe's wallet per the milestone schedule. Future contributor allocations are handled off-chain (token agreements between Knowable and contributors); once ARM is in the Safe's wallet and global transfers are enabled, Knowable distributes via standard transfers. No whitelist needed. |

### What unlocks transfers

Two paths, both irreversible:

1. **Governance proposal.** ARM holders vote to enable transfers. There are no predeclared conditions — holders decide when.
2. **Wind-down trigger.** `triggerWindDown()` automatically calls `setTransferable(true)` as a side effect. Holders must be able to move ARM to claim their pro-rata share of treasury assets — they may hold ARM on an exchange, in a multisig, or in a contract that isn't their claim address.

Both paths call the same `setTransferable(true)`. Once called, it cannot be reversed.

**Authorization:** `setTransferable(true)` is callable only by two addresses, both set immutably in the constructor:
- **Governor executor** (the timelock contract that executes passed governance proposals)
- **Wind-down contract** (calls `setTransferable(true)` as part of `triggerWindDown()`)

No other address can call this function. These addresses are constructor parameters alongside the whitelist, treasury, and `delegateOnBehalf` callers.

### What RESTRICTED means for holders

- Crowdfund participants who `claim()` receive ARM in their wallet. They can delegate and vote. They cannot transfer or sell until governance unlocks transfers.
- Team and airdrop recipients' ARM is held in the shared revenue-lock contract. Their ARM is additionally subject to revenue-gated unlock (§5.1). Even after revenue unlock, transfers remain restricted globally until governance acts.

### 5.1 Revenue-Gated Locks (Team + Airdrop only)

Team and airdrop ARM has a second restriction layer independent of the global transfer restriction. These tokens unlock based on cumulative protocol revenue:

| Cumulative Revenue | Unlocked % | Voting Power % |
|---|---|---|
| $0 | 0% | 0% |
| $10k | 10% | 10% |
| $50k | 25% | 25% |
| $100k | 40% | 40% |
| $250k | 60% | 60% |
| $500k | 80% | 80% |
| $1M | 100% | 100% |

**Key properties:**
- Revenue milestones cannot be changed by governance. The schedule is immutable.
- Revenue measurement source: **`RevenueCounter` contract** — a dedicated governance-upgradeable (UUPS) contract holding a single monotonic `recognizedRevenueUsd` counter (see GOVERNANCE.md §Revenue Counter Mechanism). Stablecoin fees are synced permissionlessly; non-stablecoin fees require a governance attestation. The revenue-lock contract reads this counter via an immutable proxy address set at deployment.
- Team/airdrop tokens gain voting power proportionally as they unlock — at $0 revenue, team and airdrop ARM has zero voting power regardless of delegation.
- Governance cannot override or accelerate team/airdrop unlocks. They unlock only via revenue.
- These restrictions are independent of the global transfer gate. A team member whose ARM is 50% revenue-unlocked still cannot transfer until governance enables global transfers.

### 5.2 Transfer restriction summary by allocation

| Allocation | Revenue-gated? | Global transfer restriction applies? | Can delegate/vote while restricted? | Transfer requires |
|---|---|---|---|---|
| Crowdfund | No | Yes | Yes (once claimed + delegated) | Global transfer unlock only |
| Team | Yes | Yes | Yes (proportional to revenue unlock) | Revenue milestone AND global transfer unlock |
| Airdrop | Yes | Yes | Yes (proportional to revenue unlock) | Revenue milestone AND global transfer unlock |
| Treasury | No | Whitelisted (can transfer at any time) | No voting power | Procedurally: governance proposal. The token contract does not enforce this — the treasury multisig signers are responsible for only executing governance-approved transfers. |

---

## 6. Voting and Delegation

ARM uses Compound-style continuous checkpointing (per GOVERNANCE.md §Voting Power).

| Property | Behavior |
|---|---|
| Voting power | Equals delegated balance at snapshot block |
| Checkpointing | Automatic on every transfer, claim, and delegation change (per GOVERNANCE.md: "Every time ARM moves — through a transfer, a claim, or a delegation change — the token contract records a new checkpoint") |
| Snapshot | `block.number - 1` at proposal creation |
| Delegation required | Yes. Undelegated ARM has zero voting power. (Per GOVERNANCE.md: "Voting power is inactive until delegated.") |
| Self-delegation | Permitted. Counts as active delegation. |
| Delegation depth | One level only. Delegates cannot redelegate. (Per GOVERNANCE.md: "A delegate cannot redelegate to a third party.") |
| Delegation change | Effective for proposals created after the next block |
| Claim-time delegation | Mandatory. See §6.1 below. |

### 6.1 Claim-Time Delegation Mechanism

Standard OZ `ERC20Votes` only lets an address delegate their own voting power — there is no built-in way for one contract to set delegation for another address. The crowdfund contract's `claim(delegate)` requires atomic transfer + delegation in a single transaction. This requires a non-standard extension on the ARM token contract.

**Specified mechanism: `delegateOnBehalf(account, delegatee)`**

```
function delegateOnBehalf(address account, address delegatee) external
```

- Callable **only** by addresses set immutably in the ARM token constructor: the crowdfund contract and the revenue-lock contract
- Sets `account`'s delegation to `delegatee`
- Emits standard `DelegateChanged(account, oldDelegatee, newDelegatee)` and `DelegateVotesChanged` events

**Crowdfund usage:** The crowdfund contract calls `ARM.transfer(participant, amount)` followed by `ARM.delegateOnBehalf(participant, delegatee)` within the same `claim()` transaction.

**Revenue-lock usage:** The revenue-lock contract calls `ARM.transfer(beneficiary, releasedAmount)` followed by `ARM.delegateOnBehalf(beneficiary, delegatee)` within the same `release(delegatee)` transaction. This ensures team/airdrop ARM is delegated immediately upon release, matching the crowdfund claim pattern. Both launch circulation paths (crowdfund claim + revenue-lock release) produce atomically delegated ARM. Treasury distributions follow a separate path without atomic delegation — see GOVERNANCE.md §Delegation-at-circulation requirement.

**Why this approach:** The alternative (requiring recipients to delegate in a separate transaction) creates a window of undelegated supply after each claim or release. For the launch circulation paths (crowdfund claim + revenue-lock release), atomic delegation ensures ARM enters governance immediately. Treasury distributions follow a different path and may create undelegated ARM until recipients delegate manually — see GOVERNANCE.md §Delegation-at-circulation requirement for the full picture. The access control is narrow: only two contracts, both set immutably at deployment, can call this function. Auditors should verify the access control cannot be changed.

### Vote-inert ARM

The following ARM has zero voting power under all circumstances:

| Category | Why | Enforcement mechanism |
|---|---|---|
| Unclaimed crowdfund ARM (still in crowdfund contract) | Not yet distributed to a holder | Crowdfund contract has no `delegate()` call path for its own balance. ARM physically in the contract cannot be delegated. |
| Undelegated ARM (held but no `delegate()` called) | Delegation is required to activate voting power | Standard ERC20Votes behavior — undelegated balance has zero voting units. |
| Treasury ARM | Governance-controlled, does not vote | **Token-enforced:** `delegate()` reverts when called by the treasury address. Hardcoded in ARM token contract. Not process discipline — the token physically prevents the treasury from delegating. |
| Unreleased team/airdrop ARM (in revenue-lock contract) | Revenue milestone not yet reached | **Architectural:** ARM sits in the lock contract which has no standalone `delegate()` call path. Unreleased ARM cannot be delegated. |
| Released team/airdrop ARM (in recipient's personal wallet) | Revenue milestone reached; ARM released | Delegated atomically at release via `release(delegatee)` → `delegateOnBehalf()`. Immediately active in governance. |

### 6.2 Voting Enforcement Architecture

Standard OZ `ERC20Votes` is sufficient for the core checkpointing. The voting restrictions do not require custom voting-unit math. They are enforced by **who holds the tokens and who can call `delegate()`**:

**Treasury non-voting:** The ARM token contract hardcodes the treasury address (constructor parameter). Any call to `delegate()` from the treasury address reverts. This is a single-line check in the `delegate()` override, not a custom voting-unit calculation. The treasury can still transfer ARM (it's whitelisted) — it just cannot convert its balance into voting power.

**Revenue-gated team/airdrop voting:** A single shared revenue-lock contract holds all team and airdrop ARM. As revenue milestones are reached, beneficiaries call `release(delegatee)` to withdraw their unlocked percentage to their personal wallet. The lock contract atomically transfers ARM and calls `delegateOnBehalf(beneficiary, delegatee)` — mirroring the crowdfund `claim(delegate)` pattern. All released ARM enters circulation delegated. The lock contract itself has no standalone `delegate()` call path — unreleased ARM remains structurally vote-inert.

This means:
- At $0 revenue: all team/airdrop ARM sits in lock contracts → zero voting power
- At 25% unlock ($50k revenue): 25% released to personal wallets → 25% delegatable
- At 100% unlock ($1M revenue): all ARM in personal wallets → full delegation possible

**Why this approach over custom voting units:** Custom `_getVotingUnits()` overrides that discount locked balances require the token contract to know about every lock contract's internal state — creating tight coupling and audit surface. The architectural approach (ARM in lock contracts cannot delegate; ARM in personal wallets can) achieves the same result with zero custom voting logic. Standard OZ `ERC20Votes` works unmodified.

### Compound-style checkpoint properties

- Flash loan attacks are structurally impossible — borrowed tokens have no checkpoint history at the snapshot block.
- Selling after voting is allowed — voting power was fixed at snapshot.
- Buying after proposal creation cannot be used on that vote.
- When tokens transfer: sender's delegated power decreases, recipient starts undelegated.

---

## 7. Minting and Supply

| Property | Value |
|---|---|
| Total supply | 12,000,000 ARM. Fixed at deployment. |
| Minting after deployment | Prohibited. No mint function exists. |
| Burning | Prohibited. No burn function exists. |
| Inflationary mechanism | None. |
| Supply cap enforcement | Enforced by absence of mint function, not by a cap variable. |

The crowdfund contract relies on pre-minted ARM only. `loadArm()` verifies `balanceOf(crowdfundContract) == MAX_SALE`; it does not mint.

---

## 8. Roles and Privileged Powers

| Role | Power | Exists at deploy? | Can be renounced? | Time-limited? | Intended disposition |
|---|---|---|---|---|---|
| Transfer gate controller | `setTransferable(true)` | Yes (governor executor + wind-down contract, both set immutably in constructor) | N/A — one-shot | No | Called once — either by governor executor (governance proposal) or by wind-down contract (side effect of triggerWindDown). Function becomes no-op after first call. |
| Revenue counter updater | Credits cumulative revenue to the milestone counter. Stablecoin fees are directly countable; non-stablecoin fees require governance attestation. | Yes (governance) | No — ongoing | No | Governance-controlled; no external oracle dependency |
| Claim/release delegation caller | `delegateOnBehalf(account, delegatee)` | Yes (crowdfund contract + revenue-lock contract) | N/A — immutable | Naturally expires when crowdfund claims and lock releases complete | Set in constructor; only these two contracts can call |
| Whitelist adder | `addToWhitelist(address)` — add-only, no removal | Yes (governance via extended proposal) | No — ongoing | No | Allows governance to whitelist new infrastructure contracts, recover from deployment address errors, or enable pre-unlock distribution channels. Cannot remove existing whitelisted addresses. |
| Admin / owner | None | No | N/A | N/A | No admin role exists |
| Minter | None | No | N/A | N/A | No minting capability |
| Pauser | None | No | N/A | N/A | No pause function |
| Upgrader | None | No | N/A | N/A | See §9 |
| Metadata role | None | No | N/A | N/A | Name and symbol are immutable |

**There is no unilateral admin role on the ARM token contract.** Privileged operations require governance (extended proposal for whitelist additions, standard for revenue attestation) or are one-shot (transfer gate). The initial whitelist is constructor-set (crowdfund, treasury, revenue-lock); governance can expand it but never shrink it. `delegateOnBehalf` callers are immutably set at deployment (crowdfund + revenue-lock). `setTransferable` callers are immutably set at deployment (governor executor + wind-down contract). No single address can modify token behavior unilaterally.

---

## 9. Upgradeability

| Property | Value |
|---|---|
| Proxy pattern | None |
| Upgrade mechanism | None |
| Deployed bytecode | Final and immutable |

The ARM token contract is not upgradeable. There is no proxy, no UUPS, no diamond, no beacon. The deployed bytecode is the permanent contract.

**Rationale:** The ARM token is the root of the system — the governor, revenue-lock contract, and all downstream contracts reference it by address. Upgradeability would make every invariant in §12 conditional on governance not voting to change them, weakening the core trust guarantee. The custom surface area beyond battle-tested OZ primitives is small (~50 lines — see below), making the probability of a critical bug low enough to accept the migration cost in a worst case over the trust cost of upgradeability. ARM does not enter the shielded pool (it's a governance/ownership token, not a payment asset), so token migration would not affect shielded notes.

**Worst-case recovery path:** Deploy a new ARM token with fixed code, snapshot balances, coordinate migration. Every contract that references the ARM token address (governor, revenue-lock, and any future integrations) would also need to be updated or redeployed to point to the new token. Disruptive but feasible — the scope of downstream updates depends on the final architecture and should be assessed once Ian settles the contract dependency graph.

---

## 10. Crowdfund Integration Assumptions

These are the exact properties the crowdfund contract depends on. Test compatibility directly.

| Assumption | Details |
|---|---|
| ARM has 18 decimals | All PARAMETER_MANIFEST.md values use 18-decimal encoding |
| Standard `transfer()` works | Crowdfund contract calls `ARM.transfer(participant, amount)` during `claim()`. Must succeed for whitelisted callers in RESTRICTED state. |
| `balanceOf()` is accurate | `loadArm()` checks `ARM.balanceOf(crowdfundContract) == MAX_SALE` |
| No fee-on-transfer | Transfer amount equals received amount. No hidden deductions. |
| No revert on zero-amount transfer | ARM follows standard OZ ERC-20 behavior: `transfer(addr, 0)` succeeds. However, the crowdfund contract's `claim()` requires `allocations[msg.sender] > 0` — zero-allocation participants claim refunds via `claimRefund()`, not `claim()`. So zero-amount ARM transfers do not arise in the crowdfund integration path. |
| Crowdfund contract is whitelisted | Can send ARM while global transfers are restricted |
| Treasury is whitelisted | Can receive ARM sweeps via `withdrawUnallocatedArm()` while restricted |
| `claim(delegate)` records delegation on ARM contract | The crowdfund contract calls `ARM.transfer(participant, amount)` then `ARM.delegateOnBehalf(participant, delegatee)` atomically within `claim()`. `delegateOnBehalf()` is callable by the crowdfund contract and the revenue-lock contract (both set immutably in constructor; see §6.1). Per GOVERNANCE.md: "ARM tokens cannot be claimed without simultaneously designating a delegatee." |
| Delegation is effective immediately | After `claim()`, the participant's ARM is delegated at the next block's checkpoint |
| Unclaimed ARM has no voting power | ARM sitting in the crowdfund contract is not delegated and contributes zero to quorum |

---

## 11. Event Surface

### Standard ERC-20 events (required)

| Event | Fields |
|---|---|
| `Transfer` | `address indexed from, address indexed to, uint256 value` |
| `Approval` | `address indexed owner, address indexed spender, uint256 value` |

### Voting/delegation events (required)

| Event | Fields | Notes |
|---|---|---|
| `DelegateChanged` | `address indexed delegator, address indexed fromDelegate, address indexed toDelegate` | Emitted on every delegation change, including claim-time delegation |
| `DelegateVotesChanged` | `address indexed delegate, uint256 previousBalance, uint256 newBalance` | Emitted when a delegate's voting power changes (from transfers, delegations, or claims) |

### Transfer restriction events

| Event | Fields | Notes |
|---|---|---|
| `TransferabilityEnabled` | — | Emitted once when governance unlocks transfers. Irreversible. |

### Revenue events

`RevenueUpdated` is **not** an ARM token event. It is emitted by the `RevenueCounter` contract when cumulative revenue is updated (see GOVERNANCE.md §Revenue Counter Mechanism and REVENUE_LOCK.md). Monitoring reads it from the `RevenueCounter` address, not from the ARM token.

---

## 12. Invariants / Forbidden Behaviors

These must hold at all times. Auditors should verify each.

| Invariant | Description |
|---|---|
| **Fixed supply** | `totalSupply()` always returns exactly 12,000,000 × 10^18. No code path changes this. |
| **No mint** | No function or code path can create new ARM. |
| **No burn** | No function or code path can destroy ARM. `totalSupply()` is constant. |
| **No blacklist** | No address can be individually blocked from holding or receiving ARM. The only transfer restriction is the global gate. |
| **No clawback** | No code path can move ARM out of a holder's wallet without their signature (transfer/approve). |
| **No pause** | No function can halt all token operations. |
| **Transfer gate is one-way** | Once `setTransferable(true)` is called, it cannot be reversed. |
| **Whitelist is add-only** | Initial whitelist is constructor-set. Governance can add addresses (extended proposal) but can never remove. Once whitelisted, always whitelisted. |
| **`delegateOnBehalf` access is immutable** | Only the crowdfund contract and the revenue-lock contract (both set immutably in constructor) can call `delegateOnBehalf()`. No function can change this. |
| **Proposal bonds inactive pre-transfer-unlock** | Governance operates on proposal threshold only (12,000 delegated ARM) while transfers are restricted. Bond mechanism activates post-transfer-unlock. |
| **Revenue schedule is immutable** | The revenue milestone table cannot be changed by governance or any admin. |
| **Revenue-lock contracts cannot delegate** | Lock contracts have no code path that calls `delegate()` on the ARM token. Unreleased team/airdrop ARM is structurally vote-inert. |
| **Delegation is required for voting** | Undelegated ARM has exactly zero voting power, regardless of balance. |
| **Treasury ARM does not vote** | ARM held by the treasury address has no voting power. `delegate()` called by the treasury address reverts — this is token-enforced, not process discipline. |
| **Token rights are uniform by allocation** | After claim and once any applicable restrictions are lifted, 1 ARM from the crowdfund is identical to 1 ARM from team allocation. No source-based discrimination in contract behavior. |

---

## 13. Open Questions

**No open questions remain.**

**Resolved:**
- Zero-amount transfers: follow standard OZ behavior (allow). `claim()` requires `allocations > 0` so the path doesn't arise. (§10)
- Whitelist mechanism: constructor-set initial list, governance can add (extended proposal) but never remove. (§5)
- Revenue measurement: governance-attested cumulative counter per GOVERNANCE.md. (§5.1)
- Claim-time delegation: `delegateOnBehalf(account, delegatee)` callable by crowdfund contract and revenue-lock contract. (§6.1)
- Release-time delegation: `release(delegatee)` on the revenue-lock contract atomically transfers + delegates, matching the crowdfund pattern. (§6.1)
- EIP-2612 `permit()`: included at launch. (§4)
- Steward delegation ban: dropped — stewards may receive delegation. (GOVERNANCE.md)
- Upgradeability: non-upgradeable. No proxy. Invariants in §12 are unconditional. (§9)

---

## 14. Audit Assumptions

| Assumption | Notes |
|---|---|
| ARM token is a separate audit scope from the crowdfund contract | Per AUDIT_HANDOFF.md §8 |
| Governance contracts (governor, timelock) are separate audit scope | See GOVERNANCE.md |
| Revenue-lock mechanism uses governance-attested counter | No external oracle dependency; contract trusts governance attestations for non-stablecoin revenue |
| Off-chain revenue accounting is out of scope | Non-stablecoin revenue attestation happens via governance proposal; the contract trusts the attested value |
| Market behavior, tokenomics, and pricing are not auditable properties | Out of scope |

---

## 15. Dependency Map

```
ARM Token Contract
  ├── consumed by: Crowdfund Contract
  │     ├── loadArm() checks balanceOf
  │     ├── claim() calls transfer + delegateOnBehalf
  │     └── withdrawUnallocatedArm() calls transfer to treasury
  ├── consumed by: Governor Contract (post-transfer-unlock only for bonds)
  │     ├── reads checkpoints for voting power (works pre-unlock)
  │     ├── reads delegation state (works pre-unlock)
  │     ├── proposal bonds require transfer (works only post-unlock)
  │     ├── calls setTransferable(true) via proposal execution
  │     └── wind-down trigger calls setTransferable(true) as side effect
  ├── consumed by: Revenue-Lock Contract (single shared contract)
  │     ├── holds all team + airdrop ARM (2.4M)
  │     ├── tracks per-beneficiary allocations internally
  │     ├── release(delegatee) calls transfer + delegateOnBehalf
  │     └── releases proportional to governance-attested revenue counter
  └── consumed by: Monitoring
        └── reads Transfer, DelegateChanged, TransferabilityEnabled events
```
