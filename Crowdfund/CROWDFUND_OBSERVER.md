# Armada Crowdfund Observer — Interface Spec

## Purpose

A read-only, real-time visualization of the Armada crowdfund: who's in, how much they committed, and who invited whom. The full invite graph and all commitment data are public on-chain from the moment they happen — this interface makes that data legible.

Built as a standalone React app. Designed to embed later as a component in the commitment UI and armada.voyage.

---

## Data Model

All data comes from on-chain events emitted by `ArmadaCrowdfund`. No backend, no server-side database — the app reads directly from chain via RPC. Client-side IndexedDB caching persists fetched event logs and ENS resolutions across sessions to avoid redundant RPC calls on reload.

### Events consumed

| Event | Fields | Triggers |
|---|---|---|
| `ArmLoaded` | — | Transitions UI from "not yet open" to commitment window open |
| `Invited` | inviter, invitee, hop, nonce | New node + edge in graph; new row in table. `nonce` is metadata (0 for direct invites, > 0 for link-based); the standalone observer does not track pending links. |
| `Committed` | address, hop, amount | Node size update; amount update in table |
| `SeedAdded` | address | New hop-0 node with edge from Armada root |
| `Finalized` | sale_size, allocated_arm, net_proceeds, refundMode | Settlement overlay / final state banner. `refundMode` distinguishes success from refund-only finalization. |
| `Allocated` | address, totalArmAmount, totalRefundAmount | Aggregate per-address settlement. Only emitted on success path (not refundMode). |
| `AllocatedHop` | address, hop, armAmount | Per-hop ARM allocation. Only emitted when armAmount > 0, success path only. Populates per-hop breakdown in row expansion and tree node detail. |
| `ArmClaimed` | address, arm_amount | Marks address as ARM-claimed in table |
| `RefundClaimed` | address, usdc_amount | Marks address as refund-claimed in table |
| `Cancelled` | — | Cancel state indicator |
| `SettlementComplete` | — | Phased fallback only. Signals all `Allocated` and `AllocatedHop` events have been emitted. Observer transitions from "settlement in progress" to final display. Under single-tx finalization, not needed — all events arrive with `Finalized`. |

Post-finalization per-address allocations come from `Allocated` (aggregate) and `AllocatedHop` (per-hop detail) events emitted during finalization — by `finalize()` in single-tx mode, or by `emitSettlement()` in phased mode. The observer does not replay the allocation algorithm client-side. **Settlement invariant:** for each address, `sum(AllocatedHop.armAmount) == Allocated.totalArmAmount`. If neither event is present for an address (refundMode or cancel), the address shows full refund eligibility derived from its `Committed` totals. Under phased settlement, the observer treats `Finalized` without `SettlementComplete` as settlement-in-progress.

### Derived state

```
nodes: Map<(address, hop), {
  address: string,
  ens: string | null,
  hop: number,
  invites_received: number,   // count of invites to THIS (address, hop) — determines cap and outgoing slots
  committed: number,          // USDC committed at this hop (capped at invites_received × HOP_CAP[hop])
  raw_deposited: number,      // total deposited (may exceed cap)
  invited_by: address[],      // all inviters at THIS hop — multiple if duplicate invites; empty for seeds
  invites_used: number,       // outgoing invite slots consumed
  invites_available: number,  // outgoing slots: invites_received × INVITE_BUDGET[hop] (3 per hop-0 slot, 2 per hop-1 slot, 0 for hop-2)
  // Post-finalization (populated from AllocatedHop events):
  allocated_arm: number | null,  // ARM allocated at this specific hop; null pre-finalization
}>

edges: Array<{
  from: (address, hop),
  to: (address, hop),
}>
```

Each invite to an `(address, hop)` creates a new participation slot — the node's effective cap equals `invites_received × HOP_CAP[hop]`, and outgoing invite slots scale proportionally. For hop-0, seeds receive one slot via `SeedAdded` (not `Invited`). The `edges` array is the canonical source of invite topology; `invited_by` on the node is a convenience accessor derived from edges.

```
// Per-address aggregate (for table and collapsed tree view)
address_summary: Map<address, {
  address: string,
  ens: string | null,
  hops: number[],             // which hops this address occupies
  total_committed: number,    // sum of committed across all hops
  per_hop: Map<hop, number>,  // committed amount per hop
  display_inviter: string,    // see Display Inviter Rule below
  // Post-finalization (populated from Allocated + AllocatedHop events):
  allocated_arm: number | null,          // total ARM (from Allocated event)
  allocated_per_hop: Map<hop, number>,   // ARM per hop (from AllocatedHop events)
  refund_usdc: number | null,            // total refund (from Allocated event)
  // Claim status (populated from ArmClaimed / RefundClaimed events):
  arm_claimed: boolean,
  refund_claimed: boolean,
}>
```

**Display inviter rule:** A multi-hop address may have different inviters at each hop (e.g., self-invited at hop-1 but invited by a different seed's wallet at hop-2). For collapsed display in the table and tree tooltip:
- If the address occupies hop-0: `display_inviter = "Armada"` (seed)
- If the address has the same inviter at all hops: `display_inviter = that_inviter`
- If the address has different inviters at different hops: `display_inviter = inviter_at_lowest_hop` with a "multiple inviters" indicator; full per-hop breakdown shown on row expansion

### ENS Resolution

Resolve ENS names lazily — batch-resolve on initial load, then resolve new addresses as `Invited`/`Committed` events arrive. Cache in IndexedDB (see Data Persistence below). Display ENS name as primary label with address as tooltip; fall back to truncated address (`0x1234…abcd`) if no ENS.

---

## Views

The interface has two linked views (tree and table) and a persistent stats banner. Clicking/searching an address in either view highlights it in both.

### Stats Banner

Always visible at top. Updates in real time.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Hop-0: $987,000 (131% of ceiling)    Hop-1: $342,000 (67% of ceiling) │
│  Hop-2: $48,000 (80% of floor)                                         │
│  Capped Demand: $1,377,000    Sale Size: BASE ($1.2M)    Status: OPEN  │
│  Seeds: 142/150    Participants: 847    Time remaining: 6d 14h          │
└──────────────────────────────────────────────────────────────────────────┘
```

**Fields:**
- Per-hop committed totals. Hop-0 and hop-1 show oversubscription as percentage of their enforced ceiling — when >100%, a pro-rata scaling indicator appears (e.g., "131% → ~76% fill"). Hop-2 shows **floor coverage** as percentage of its reserved floor — >100% means hop-2 demand exceeds the floor, but this does NOT imply pro-rata scaling (hop-2's effective ceiling grows with hop-1 leftover, so >100% of floor is not necessarily oversubscribed).
- Total `capped_demand` (sum of all per-address per-hop capped balances — this is the number that determines expansion and minimum raise)
- Current sale size (BASE or EXPANDED) — updates to EXPANDED when capped_demand crosses $1.5M
- Status: OPEN / AWAITING FINALIZATION / FINALIZED / CANCELLED / REFUND MODE (no `FINALIZING` state — finalization is a single atomic transaction)
- Seed count (used / cap)
- Total distinct participants (unique addresses, not nodes)
- Countdown to commitment deadline

### Tree View

A layered DAG visualization with four vertical bands: Armada (ROOT), hop-0, hop-1, hop-2. Left to right.

**Node rendering:**

Each `(address, hop)` pair is a node. Nodes are circles, sized by committed amount at that specific (address, hop) position. Minimum diameter ensures hop-2 nodes ($1k max) remain clickable. Size scale is sqrt-based (not linear) so the hop-0 → hop-2 differential is visible without making hop-2 invisible.

**Multi-hop addresses:**

When an address appears at multiple hops, it renders as a **single node at its highest-trust hop** (hop-0 if present, else hop-1). The node is sized by the address's **total commitment across all hops.** A segmented fill shows the per-hop breakdown — three colored bands proportional to per-hop amounts:

- Hop-0 segment: base color
- Hop-1 segment: lighter shade
- Hop-2 segment: lightest shade

A single-hop node has solid fill. Multi-hop nodes are visually distinct at a glance by their banded appearance.

**Expanding multi-hop nodes:** Clicking a multi-hop node expands it into its constituent `(address, hop)` positions, showing the invite edges for each. Not all multi-hop edges are self-invites — an address can appear at hop-0 (as a seed, invited by Armada) and at hop-2 (invited by an unrelated hop-1 participant). The expanded view shows each position's actual inviter. Clicking again collapses. A small indicator (hop count badge: "×3" or "×2") signals expandability before clicking.

**Edges:**

Directed lines from inviter to invitee. Color-coded by hop transition (hop-0→1, hop-1→2). When a multi-hop node is expanded, self-invite edges (same address inviting itself to a different hop) are shown as curved loops; cross-entity edges show the actual inviter.

**ROOT node:**

Labeled "Armada" with a distinct visual treatment (protocol icon or logo mark, not a circle). Edges from ROOT to seeds (hop-0) and to launch-team hop-1/hop-2 placements. Launch team placements have a subtly different edge style (dashed or different color) to distinguish them from seed-originated invites.

**Layout:**

- Four vertical columns: ROOT | Hop-0 | Hop-1 | Hop-2
- Within each column, nodes are arranged vertically, grouped by inviter subtree where possible
- When a subtree is too large to display inline, it collapses with a count indicator ("12 invitees") expandable on click
- Panning and zooming via mouse/touch — the full graph at 150 seeds will not fit on one screen

**Empty state:** Before any seeds are added, the view shows only the Armada ROOT node with a "Waiting for seeds…" indicator.

### Table View

A sortable, searchable, filterable data table.

**Columns:**

| Column | Content | Sortable | Notes |
|---|---|---|---|
| Address | ENS name or truncated address | Yes (alpha) | Full address on hover/click. Link to block explorer. |
| Hop(s) | Hop levels this address occupies | Yes | Shows all hops: "0, 1, 2" for full multi-hop. Filter dropdown. |
| Committed | Total USDC committed across all hops | Yes (default: desc) | Expandable to show per-hop breakdown inline |
| Invited by | Display inviter (see rule above) | Yes | "Armada" for seeds/launch-team. Shows lowest-hop inviter for multi-hop; "multiple" indicator if inviters differ. Full breakdown on expansion. |
| Invites | Used / available | Yes | Combined outgoing slots across all hops. e.g., "4/9" for a seed at hop-0 (3 slots) who self-invited 3× to hop-1 (gaining 6 hop-2 slots). Scales with invites received. |
| Allocated | ARM allocated (post-finalization only) | Yes | Hidden pre-finalization. Shows allocated ARM and refund USDC. |
| Claimed | Claim status icons | No | Post-finalization only. Two indicators: ARM claimed ✓/✗, refund claimed ✓/✗. Hidden pre-finalization. |

**Default sort:** Descending by total committed amount.

**Search:** Text input that filters by address or ENS name. Matches partial strings. Results update as the user types (debounced). Searching highlights the matching node(s) in the tree view simultaneously.

**Filters:**
- Hop level (multi-select: show hop-0 only, hop-1 only, etc.)
- Multi-hop only (toggle: show only addresses at 2+ hops)
- Oversubscribed hops only (toggle)

**Row expansion:** Clicking a row expands to show per-hop detail: amount committed at each hop, cap remaining, inviter at each hop (if different), invitees list with their amounts. Post-finalization: shows per-hop ARM allocation (from `AllocatedHop`) and aggregate refund amount (from `Allocated`). This is the table equivalent of expanding a multi-hop node in the tree.

**Address cross-linking:** Clicking an address in the "Invited by" column scrolls to and highlights that inviter's row. Clicking any address highlights the corresponding node in the tree view.

---

## Interaction Between Views

The tree and table are two views of the same data, displayed simultaneously (side by side on desktop, tabbed on mobile). All interactions cross-link:

| Action | Tree effect | Table effect |
|---|---|---|
| Click node in tree | Select + highlight node | Scroll to and highlight row |
| Click row in table | Pan to and highlight node | Select row |
| Search in table | Pan to and highlight matching node(s) | Filter to matching rows |
| Expand multi-hop in tree | Show per-hop nodes and edges | Expand corresponding row |
| Hover node in tree | Show tooltip with summary | Highlight row (subtle) |

Selection state is shared — at most one address is "selected" at a time, shown in both views.

---

## Data Fetching

### Initial load

On page load, check IndexedDB for cached event logs. Fetch only new events since the last cached block. On first visit, fetch all historical events from contract deployment to current block via batch RPC call (`eth_getLogs`). Build the full graph and table state.

### Live updates

**Phase 1 (ship first):** Poll every 15 seconds via `eth_getLogs` for new events since last fetched block. Merge into existing state. Persist new events to IndexedDB. Animate new nodes/edges into the tree (fade in, grow from zero size). Update stats banner.

**Phase 2 (upgrade later):** WebSocket subscription to contract events. Same merge logic, lower latency. The data layer is abstracted behind an `EventSource` interface so swapping poll → websocket requires no changes to the view layer.

### Data persistence (IndexedDB)

Client-side persistence via IndexedDB for two categories:
- **Event log cache:** All fetched contract events, keyed by block number. Eliminates redundant RPC calls on page reload. Invalidation: none needed — events are append-only and immutable once confirmed.
- **ENS name cache:** Resolved ENS mappings, keyed by address. TTL: 24 hours (ENS names can change, but not frequently enough to warrant per-session resolution).

No user data is stored. Cache is purely an optimization — the app functions identically on a cold start with an empty cache, just slower.

### RPC provider

Configurable with ordered fallback list — try primary, fall back to secondary on failure. Default to a reliable public RPC endpoint. No API key required (read-only calls only). The provider abstraction expects multiple RPCs from day one: if the primary is slow or rate-limited, the next provider in the list is tried automatically. The app should work with any EVM-compatible RPC endpoint.

---

## State Transitions

The interface reflects the contract's state machine. There is no `FINALIZING` state — finalization is a single atomic transaction; the UI transitions directly from "awaiting finalization" to "finalized." (If phased finalization is implemented as a gas fallback, the UI shows "Settlement in progress — allocation data arriving" between `Finalized` event and `SettlementComplete`. See CROWDFUND.md Gas Considerations.)

| Contract state | UI behavior |
|---|---|
| Pre-commitment (ARM not loaded) | "Crowdfund not yet open" — empty state |
| Commitment window open | Full tree + table, live updates, stats banner with countdown |
| Commitment deadline passed, not finalized | Stats banner shows "Awaiting finalization." Tree and table remain. If capped_demand < MIN, show "Below minimum raise — refunds available." |
| Finalized (success) | Stats banner shows final figures. "FINALIZED" badge. Allocated column appears in table showing total ARM allocated and USDC refund per address (from `Allocated` events). Row expansion shows per-hop ARM breakdown (from `AllocatedHop` events). Claimed column tracks claim status as `ArmClaimed`/`RefundClaimed` events arrive. Tree nodes gain an allocation indicator (ring or label showing allocated vs. committed). |
| Finalized (refundMode) | "REFUND MODE" indicator. No `Allocated` or `AllocatedHop` events emitted. All addresses show "Full refund available" derived from `Committed` totals. Claimed column tracks refund claims only. |
| Cancelled | "CANCELLED" indicator. No allocations. "Full refund available." Claimed column tracks refund claims. |

---

## Visual Design Direction

The aesthetic should match armada.voyage — the one-pager's palette and typography are the reference point. Dark background. Monospace for addresses and numbers. Sans-serif for labels. The graph should feel like a living network visualization, not a static org chart.

Color semantics:
- Hop-0: primary accent (the core)
- Hop-1: secondary shade (one step out)
- Hop-2: tertiary/muted (the periphery)
- Oversubscribed: warm warning color on the relevant hop
- Multi-hop indicator: segmented fill using the hop colors above
- Armada ROOT: distinct from all hops — protocol-level color

Node hover tooltip:
```
alice.eth (0x1234…abcd)
Hop-0: $15,000 committed
Hop-1: $4,000 committed (1 slot)
Total: $19,000
Invited by: Armada (seed) · hop-1: self-invited
Invites: 1/5 used

[Post-finalization, if applicable:]
Allocated: 18,600 ARM
Refund: $400
ARM claimed: ✓   Refund claimed: ✗
```

---

## Technical Architecture

```
src/
  components/
    StatsBar.tsx          // live stats banner
    TreeView.tsx          // DAG visualization (d3)
    TableView.tsx         // sortable/searchable table
    NodeDetail.tsx        // expanded node/row detail
    SearchBar.tsx         // shared search input
  hooks/
    useContractEvents.ts  // event fetching + polling + IndexedDB cache
    useGraphState.ts      // derived graph state from events
    useENS.ts             // ENS resolution with IndexedDB cache
    useSelection.ts       // shared selection state between views
  lib/
    events.ts             // event parsing and type definitions
    graph.ts              // graph construction from events
    cache.ts              // IndexedDB read/write helpers
    rpc.ts                // multi-provider with ordered fallback
    constants.ts          // contract address, ABI fragments, hop params
  App.tsx                 // layout: stats bar + tree/table split
```

### Key dependencies

- **React** — component framework
- **d3-hierarchy + d3-force** — layered DAG layout (strict depth structure favors d3 over freeform graph libraries)
- **viem** or **ethers** — RPC interaction and event parsing
- **@tanstack/react-table** — table with sort/filter/search
- **wagmi** — ENS resolution hooks (read-only, no wallet connection needed)
- **idb** — lightweight IndexedDB wrapper for event and ENS caching

### Performance targets

- Initial load: < 3s for full event history at ~2,000 events (150 seeds × ~10 events each)
- Live update: < 500ms from event to render
- Smooth interaction at 1,740 nodes (full network) — may require virtualized rendering or canvas-based graph at scale

---

## Scope Boundaries

**In scope:**
- Read-only visualization of on-chain invite graph and commitment data
- Post-finalization allocation and refund display (aggregate from `Allocated` events; per-hop ARM breakdown from `AllocatedHop` events)
- Claim status tracking (ARM claimed, refund claimed)
- ENS resolution with IndexedDB persistence
- Real-time updates (polling, upgradable to websocket)
- Search, sort, filter
- Cross-linked tree and table views
- Client-side IndexedDB caching for event logs and ENS
- Multi-RPC provider with ordered fallback
- Responsive (desktop-first, functional on mobile)

**Out of scope (future work):**
- Wallet connection / commitment submission (separate commitment UI)
- Allocation preview ("what would I get if I committed X") — requires running the allocation algorithm client-side
- Historical playback ("show me the graph at block N")
- Export (CSV, image)
- Authentication or access control (fully public, no login)
- Backend / server-side indexer (all data from RPC + client-side cache)

**Deferred but designed for:**
- Embedding as a component in the commitment UI (the tree and table components accept data via props, not internal fetching — the data layer is separable)
- WebSocket upgrade (EventSource interface abstraction)
