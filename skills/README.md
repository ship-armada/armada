# Armada Protocol — Contributor Memo

**To:** All Contributors  
**From:** Armada core team (via Claude)  
**Date:** March 15, 2026  
**Re:** Building Armada Skills — AI Dev Tooling for the Protocol

---

> **TL;DR:** Circle and the Ethereum community ship protocol-specific "skills" — SKILL.md files that give AI coding assistants correct priors about their protocol. We need to do the same. Armada's privacy invariants are too subtle and too consequential for LLMs to get right by default.

https://github.com/circlefin/skills 
https://ethskills.com/SKILL.md 

---

## What are skills and why does this matter?

Skills are markdown files (SKILL.md) that developers load into AI coding assistants — Claude Code, Cursor, Windsurf — before writing protocol-specific code. They encode decision frameworks, correct patterns, and common mistakes so the agent starts from accurate mental models rather than guessing.

Circle ships skills for CCTP, wallets, and smart contracts. The Ethereum community ships skills for gas costs, ERC standards, and security patterns. Both exist because LLM training data is stale and incomplete — especially for niche protocol details.

For Armada, the stakes are higher. Circle's skills prevent bugs. Ours would prevent privacy failures, which are irreversible. A contributor who misuses the shielded pool API, bypasses the paymaster, or mixes Standard and Fast Transfers doesn't produce wrong output — they deanonymize users.

---

## The four skills we're building

| Skill file | What it encodes |
|---|---|
| `armada-privacy/SKILL.md` | ZK shielding lifecycle, anonymity set discipline, Standard vs Fast Transfer invariants, undetectability requirements |
| `armada-cctp/SKILL.md` | CCTPHookRouter architecture, destinationCaller enforcement, burn message expiration, reattest handling, Iris API rate limits |
| `armada-paymaster/SKILL.md` | Gas abstraction flow, fee routing, treasury hardcoding, why paymasters are the revenue capture point |
| `armada-integrator/SKILL.md` | How external devs build on Armada — correct entry points, what not to bypass, lifecycle guarantees |

---

## What each skill must cover

### armada-privacy

- The full shielding lifecycle: shield → internal transfers → unshield. Skipping steps is not supported and breaks privacy guarantees.
- Standard Transfer exclusively. Fast Transfer fragments the anonymity set — never mix them.
- Undetectability as a first-class constraint: every on-chain artifact must be indistinguishable from baseline Ethereum activity.
- Why encrypting hookData matters: NPK traveling in plaintext enables entry-graph construction by chain analysis firms even without breaking the ZK layer.

### armada-cctp

- `CCTPHookRouter` is architecturally load-bearing, not optional middleware — it provides both gas abstraction and atomic hook dispatch.
- `destinationCaller` must be enforced to prevent third-party relayer racing. This is a correctness requirement, not an optimization.
- Burn messages expire after 24 hours. `iris-relay.ts` must handle reattest flows, not assume messages are always live.
- Circle's CCTP V2 does not natively auto-dispatch hooks. The relayer does. This distinction matters for every integration.
- Per-chain fee queries and Iris API rate-limit backoff/queuing are required for production reliability.

### armada-paymaster

- The paymaster is the primary revenue capture point. At current gas costs it can capture near-total margin on gas abstraction.
- Fee routing to treasury is hardcoded on-chain. This is intentional and must not be modified.
- Users pay fees in shielded USDC. The paymaster handles conversion — integrators do not route fees themselves.

### armada-integrator

- Entry point is the `PrivacyPoolClient` contract on the client chain. Do not interact with hub contracts directly.
- Hub-and-spoke topology: Ethereum is the hub, client chains run lightweight contracts. Cross-chain logic routes through CCTP.
- The paymaster must be used for transaction submission — bypassing it breaks fee accounting and anonymity set integrity.

---

## Format and publishing

Follow the pattern from ethskills.com: a root `SKILL.md` acts as a routing index that tells the agent which sub-skill to fetch for a given task. Each sub-skill is standalone — agents load only what they need.

The root index should include a task routing table:

| I'm working on... | Fetch this skill |
|---|---|
| CCTP relayer code | `armada-cctp/SKILL.md` |
| Building an integrator app | `armada-integrator/SKILL.md` |
| Paymaster or fee logic | `armada-paymaster/SKILL.md` |
| Any shielding pool changes | `armada-privacy/SKILL.md` |

Publish the skills repo publicly on GitHub under the Armada organization and reference it in the developer docs. The Circle CCTP `bridge-stablecoin` skill is a useful prerequisite — our skills should reference it rather than duplicate CCTP fundamentals.

Skills are designed to stay stable even when slightly behind. For details that change frequently (contract addresses, chain IDs, API endpoints), we'll complement with an MCP server once the protocol is live.

---

## Why now

Pre-launch is the correct time to write skills — it forces us to specify precisely where the protocol enforces invariants versus where it trusts callers. That discipline is valuable documentation independent of AI tooling.

When our first integrators start building, they'll use Claude Code or Cursor. Without skills, those tools will generate code that ignores anonymity set discipline or routes around the paymaster entirely. With skills, we ship correct priors directly into their development environment — a developer experience advantage that's hard to replicate.
