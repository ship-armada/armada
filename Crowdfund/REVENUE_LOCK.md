# Revenue-Lock Contract — Behavior Spec

## 1. Purpose

A single shared contract that holds all team and airdrop ARM (2,400,000 total) and releases it to beneficiaries as cumulative protocol revenue milestones are reached. This is the enforcement mechanism for revenue-gated token unlocks described in GOVERNANCE.md and ARM_TOKEN.md.

**This is a contract behavior spec.** It defines what the contract does, what it reads, and what it guarantees.

---

## 2. Architecture

```
RevenueCounter (UUPS proxy, governance-upgradeable)
  └── recognizedRevenueUsd() → uint256 (monotonic)

RevenueLock (immutable, non-upgradeable)
  ├── reads: RevenueCounter.recognizedRevenueUsd()
  ├── holds: 2,400,000 ARM (team + airdrop)
  ├── tracks: per-beneficiary allocations and released amounts
  └── releases: ARM to beneficiary wallet + delegates atomically via delegateOnBehalf()

ARM Token
  ├── whitelist includes RevenueLock address (constructor-set)
  └── delegateOnBehalf() callable by RevenueLock (constructor-set)
```

---

## 3. Deployment

### Constructor parameters

| Parameter | Value | Notes |
|---|---|---|
| ARM token address | Immutable | Must match the ARM token that whitelists this contract |
| RevenueCounter address | Immutable | UUPS proxy address — implementation may be upgraded by governance, but the proxy address never changes |
| Beneficiary list | Immutable | Array of `(address beneficiary, uint256 allocationAmount)` pairs. Set once at deployment. |
| Milestone table | Immutable | The revenue-to-unlock-percentage mapping. Hardcoded or constructor-set. Cannot be changed after deployment. |

### Post-deployment setup

1. ARM token constructor mints 2,400,000 ARM directly to this contract's address (no intermediate deployer step)
2. Verify `ARM.balanceOf(revenueLockContract) == 2_400_000e18`

### Deployment order

There are circular dependencies across the contract set. The ARM token constructor needs immutable references to: revenue-lock (whitelist + `delegateOnBehalf`), crowdfund (whitelist + `delegateOnBehalf`), treasury (whitelist + delegation revert), governor executor / timelock (`setTransferable` caller), and wind-down contract (`setTransferable` caller). The revenue-lock constructor needs the ARM token address and the RevenueCounter address. The governor/timelock may need the ARM token address for voting power reads.

**Recommended: CREATE2 precomputed addresses.** Compute all contract addresses before deploying any of them. The full set that may need precomputation: ARM token, RevenueLock, RevenueCounter proxy, Governor/Timelock, WindDown contract, and CrowdfundContract. Deploy in any order — all constructors use the precomputed addresses.

This avoids initialization functions and keeps all immutable contracts fully configured from deployment. Ian should confirm the exact dependency graph and CREATE2 approach during implementation.

---

## 4. Beneficiary List

Set at deployment. Cannot be modified after deployment.

| Beneficiary | Allocation | Notes |
|---|---|---|
| Team member 1 | `[amount]` | Individual |
| Team member 2 | `[amount]` | Individual |
| ... | ... | ... |
| Knowable Safe | `[amount]` | Larger allocation for future contributors. Released ARM distributed off-chain via token agreements after global transfer unlock. |
| Airdrop recipient 1 | `[amount]` | Individual |
| ... | ... | ... |

**Total must equal exactly 2,400,000 ARM** (1,800,000 team + 600,000 airdrop). The contract should verify this at deployment.

**No post-deployment modifications.** There is no function to add, remove, or change beneficiaries. There is no admin role. The Knowable Safe handles future contributor allocation off-chain after its ARM is released and global transfers are enabled — the lock contract doesn't need to know about this.

---

## 5. Milestone Table

The revenue-to-unlock mapping. Immutable after deployment.

| Cumulative Revenue (USD) | Unlocked % |
|---|---|
| $0 | 0% |
| $10,000 | 10% |
| $50,000 | 25% |
| $100,000 | 40% |
| $250,000 | 60% |
| $500,000 | 80% |
| $1,000,000 | 100% |

**No interpolation.** Unlock percentage is a step function — at $49,999 revenue, unlock is 10%; at $50,000, it jumps to 25%.

**No time-based fallback.** If revenue never reaches $1M, tokens never fully unlock. There is no calendar-based override.

---

## 6. Release Mechanism

### `release(address delegatee)`

The only state-changing function a beneficiary calls.

**Preconditions:**
- `msg.sender` is a beneficiary in the list
- `delegatee != address(0)`

**Logic:**
1. Read `RevenueCounter.recognizedRevenueUsd()`
2. Look up the unlock percentage from the milestone table (step function)
3. Compute entitled amount: `unlockPercentage × allocation[msg.sender]`
4. Compute releasable amount: `entitled - alreadyReleased[msg.sender]`
5. If releasable == 0: revert (nothing new to release)
6. Update `alreadyReleased[msg.sender] += releasable`
7. Call `ARM.transfer(msg.sender, releasable)` — succeeds because this contract is whitelisted
8. Call `ARM.delegateOnBehalf(msg.sender, delegatee)` — succeeds because this contract is an authorized caller

**Events:**
```
Released(address indexed beneficiary, uint256 amount, address delegatee, uint256 cumulativeReleased)
```

**Properties:**
- **Pull-based.** Beneficiaries call `release()` when they want. There is no push mechanism. ARM sits in the contract until claimed.
- **Atomic transfer + delegation.** Released ARM enters circulation already delegated, matching the crowdfund `claim(delegate)` pattern. No ARM enters circulation undelegated.
- **Delegation applies to full balance.** Step 8 calls `delegateOnBehalf(msg.sender, delegatee)`, which sets the beneficiary's entire ARM delegation — not just the newly released tokens. This matches standard ERC20Votes behavior (one delegatee per address). If the beneficiary previously delegated to Alice and now specifies Bob, all their ARM (including previously released tokens) delegates to Bob.
- **Idempotent.** Calling `release()` when nothing new is unlocked reverts harmlessly. Calling it multiple times between milestones is safe — `alreadyReleased` tracks cumulative releases.
- **Delegatee can be changed.** Each `release()` call sets delegation to the provided `delegatee`. A beneficiary can also change their delegatee at any time by calling `ARM.delegate()` directly on their wallet — no need to wait for the next milestone.

---

## 7. View Functions

| Function | Returns | Notes |
|---|---|---|
| `allocation(address beneficiary)` | uint256 | Total allocation for this beneficiary |
| `released(address beneficiary)` | uint256 | Total ARM already released to this beneficiary |
| `releasable(address beneficiary)` | uint256 | ARM currently available to release (entitled - released) |
| `unlockPercentage()` | uint256 (bps) | Current unlock percentage based on revenue counter |
| `currentRevenue()` | uint256 | Current `recognizedRevenueUsd` from the counter |
| `beneficiaryCount()` | uint256 | Number of beneficiaries |

---

## 8. What This Contract Does NOT Do

| Absent capability | Why |
|---|---|
| No admin role | Beneficiary list and milestone table are immutable |
| No `addBeneficiary()` or `removeBeneficiary()` | List is set at deployment and cannot change |
| No `reassign()` | Future contributor allocation is handled off-chain by Knowable after release |
| No `delegate()` on behalf of unreleased tokens | Unreleased ARM sits in this contract; this contract has no delegation code path for its own balance — unreleased ARM is structurally vote-inert |
| No upgradeability | No proxy, no UUPS. Deployed bytecode is final. |
| No wind-down interaction | If wind-down triggers, this contract is unaffected. Unreleased ARM stays locked. Beneficiaries can still call `release()` if milestones have been reached. If the protocol never earned enough revenue, the tokens simply stay here. |
| No `withdrawAll()` or sweep function | ARM can only leave via `release()` to the entitled beneficiary. No backdoor. |

---

## 9. Invariants

| Invariant | Description |
|---|---|
| **Supply conservation** | `ARM.balanceOf(this) + sum(alreadyReleased[all beneficiaries]) == totalAllocation` at all times |
| **Monotonic releases** | `alreadyReleased[beneficiary]` never decreases |
| **Milestone table is immutable** | No code path modifies the unlock schedule after deployment |
| **Beneficiary list is immutable** | No code path adds, removes, or modifies beneficiaries after deployment |
| **Unreleased ARM is vote-inert** | This contract has no `delegate()` call path for its own balance. ARM sitting here has zero voting power. |
| **Released ARM is always delegated** | Every `release()` call atomically delegates via `delegateOnBehalf()`. No ARM enters circulation undelegated. |
| **No over-release** | `alreadyReleased[beneficiary] <= allocation[beneficiary]` at all times |
| **Revenue counter is monotonic** | The counter can only increase. A decrease would require a bug in the RevenueCounter contract (separate audit scope). |

---

## 10. Wind-Down Interaction

If `triggerWindDown()` is called on the wind-down contract:

- This contract is **unaffected**. It has no wind-down awareness.
- Unreleased ARM stays locked. If revenue milestones haven't been reached, the tokens remain here permanently.
- Already-released ARM is in beneficiaries' wallets and is part of the "circulating" supply eligible for pro-rata treasury distribution.
- Wind-down automatically enables global ARM transfers (`setTransferable(true)`), so beneficiaries who have released ARM can move it to claim their treasury share.

**This is the intended fairness property:** those who paid (crowdfund participants) have priority in failure scenarios. Team/airdrop tokens only unlock if the protocol earns revenue — if it failed before earning revenue, those tokens stay locked and have no claim on remaining assets.

---

## 11. Custom Grants (Future)

**The revenue-gated lock mechanism is a one-time launch construct.** There are no future custom lock contracts with revenue gating or independent milestone tables.

After governance enables global transfers, custom grants are standard treasury transfers via governance proposal. The recipient receives ARM in their wallet and delegates via standard `delegate()`. No lock contract, no atomic delegation, no whitelisting needed.

This simplifies the system:
- No registry of lock contracts for wind-down denominator calculations
- No need to extend `delegateOnBehalf` authorization beyond the two launch contracts
- The wind-down redemption denominator can hardcode the four known excluded addresses (treasury, shared revenue-lock, crowdfund, redemption contract) permanently

---

## 12. Deployment Checklist

| Check | Status |
|---|---|
| CREATE2 addresses precomputed for all contracts with circular dependencies (ARM, RevenueLock, RevenueCounter, Governor/Timelock, WindDown, Crowdfund) | ☐ |
| Beneficiary list finalized and published | ☐ |
| Total allocations sum to exactly 2,400,000 ARM | ☐ |
| Milestone table matches GOVERNANCE.md | ☐ |
| RevenueCounter proxy deployed at precomputed address | ☐ |
| RevenueLock deployed at precomputed address with correct constructor parameters | ☐ |
| ARM token deployed — constructor mints directly to this contract, crowdfund, and treasury | ☐ |
| ARM token constructor includes this contract's address in whitelist | ☐ |
| ARM token constructor includes this contract's address in `delegateOnBehalf` caller list | ☐ |
| `ARM.balanceOf(this) == 2_400_000e18` verified | ☐ |
| `release()` tested on testnet with mocked revenue counter | ☐ |

---

## 13. Dependency Map

```
RevenueLock
  ├── reads: RevenueCounter.recognizedRevenueUsd() (UUPS proxy, fixed address)
  ├── calls: ARM.transfer(beneficiary, amount) (whitelisted sender)
  ├── calls: ARM.delegateOnBehalf(beneficiary, delegatee) (authorized caller)
  ├── consumed by: Monitoring (reads Released events from this contract)
  └── consumed by: ARM_TOKEN.md §6.2 (voting enforcement architecture)

RevenueCounter (separate contract, UUPS proxy)
  ├── emits: RevenueUpdated(cumulativeRevenue, previousRevenue)
  ├── consumed by: RevenueLock (reads recognizedRevenueUsd())
  └── consumed by: Monitoring (reads RevenueUpdated events)
```
